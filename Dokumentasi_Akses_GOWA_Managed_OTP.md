# Dokumentasi Akses API GOWA Managed WhatsApp OTP Verification Service

Dokumentasi ini menjelaskan cara aplikasi client mengakses layanan **Managed WhatsApp OTP Verification Service** pada GOWA.

Layanan ini menggunakan konsep **centralized OTP service**, yaitu seluruh proses OTP dikelola oleh GOWA:

- Generate OTP
- Penyimpanan session OTP
- Pengiriman OTP melalui inbound WhatsApp no-cost flow
- Verifikasi OTP
- Verifikasi kepemilikan nomor WhatsApp
- Expiry session
- Expiry OTP
- Attempt limit
- Session lifecycle

Aplikasi client hanya perlu melakukan dua proses:

1. Request OTP
2. Verify OTP

Tidak ada callback URL dan tidak ada fallback ke aplikasi client.

---

## 1. Base URL

```text
https://gowa.gresikkab.go.id/api
```

---

## 2. Autentikasi API

Seluruh endpoint OTP yang diakses oleh aplikasi client wajib membawa **Bearer Token**.

Bearer Token ini adalah **Access Token aplikasi client** yang divalidasi oleh middleware GOWA.

> Penting: Bearer Token berbeda dengan token OTP session.
>
> - **Bearer Access Token**: token untuk akses API GOWA.
> - **OTP Session Token**: token yang dikembalikan oleh endpoint `/otp/request` dan digunakan pada `/otp/verify`.

### Header Wajib

```http
Authorization: Bearer {ACCESS_TOKEN}
Accept: application/json
Content-Type: application/json
```

Contoh:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6...
Accept: application/json
Content-Type: application/json
```

### Response Jika Bearer Token Tidak Dikirim

```json
{
  "success": false,
  "message": "Unauthenticated."
}
```

HTTP Status:

```text
401 Unauthorized
```

### Response Jika Bearer Token Tidak Valid

```json
{
  "success": false,
  "message": "Invalid bearer token."
}
```

HTTP Status:

```text
401 Unauthorized
```

### Response Jika Bearer Token Tidak Aktif / Tidak Diizinkan

```json
{
  "success": false,
  "message": "Token is inactive or not allowed."
}
```

HTTP Status:

```text
403 Forbidden
```

---

## 3. Ringkasan Flow OTP

```text
Client App
   |
   | POST /otp/request
   | Body: app_id, user_id
   v
GOWA
   |
   | Generate token session
   | Generate OTP
   | Simpan session OTP ke cache
   | Return wa_link
   v
Client App
   |
   | Tampilkan / buka wa_link
   v
User WhatsApp
   |
   | Kirim pesan:
   | Permintaan Kode #OTP{token}
   v
GOWA Inbound WhatsApp
   |
   | Validasi token
   | Bind nomor WhatsApp pengirim
   | Set status OTP_SENT
   | Kirim OTP ke WhatsApp user
   v
Client App
   |
   | User input OTP
   | POST /otp/verify
   v
GOWA
   |
   | Verifikasi OTP
   | Hapus session jika sukses
   v
Client App
```

---

## 4. Status Lifecycle OTP

| Status | Keterangan |
|---|---|
| `WAITING_WA` | Session OTP sudah dibuat, tetapi user belum mengirim pesan WhatsApp dari link OTP. |
| `OTP_SENT` | User sudah mengirim pesan WhatsApp, nomor WhatsApp sudah terikat, dan OTP sudah dikirim ke WhatsApp user. |
| `VERIFIED` | OTP berhasil diverifikasi. Session cache langsung dihapus. |
| `BLOCKED` | OTP diblokir karena attempt maksimal tercapai. Session cache langsung dihapus. |
| `EXPIRED` | Session atau OTP sudah kedaluwarsa. Session akan hilang melalui TTL cache. |

---

## 5. Endpoint Request OTP

Endpoint ini digunakan oleh aplikasi client untuk meminta session OTP baru.

```http
POST /otp/request
```

Full URL:

```text
https://gowa.gresikkab.go.id/api/otp/request
```

### Header

```http
Authorization: Bearer {ACCESS_TOKEN}
Accept: application/json
Content-Type: application/json
```

### Request Body

```json
{
  "app_id": "40118077-f9e5-4a42-82ee-af12aae2f2f5",
  "user_id": "1"
}
```

### Field Request

| Field | Tipe | Wajib | Keterangan |
|---|---:|:---:|---|
| `app_id` | string | Ya | UUID aplikasi yang terdaftar di GOWA. |
| `user_id` | string | Ya | ID user dari aplikasi client. Nilai ini digunakan sebagai identifier user pada aplikasi client. |

### Contoh cURL

```bash
curl -X POST "https://gowa.gresikkab.go.id/api/otp/request" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "40118077-f9e5-4a42-82ee-af12aae2f2f5",
    "user_id": "1"
  }'
```

### Success Response

```json
{
  "success": true,
  "token": "FfjFwJIPhJYvCKMtLNA7YIe-9DJi9LvpgHhfsSa3q-c",
  "wa_link": "https://wa.me/6281217889986?text=Permintaan+Kode+%23OTPFfjFwJIPhJYvCKMtLNA7YIe-9DJi9LvpgHhfsSa3q-c",
  "text": "Permintaan Kode #OTPFfjFwJIPhJYvCKMtLNA7YIe-9DJi9LvpgHhfsSa3q-c",
  "expires_in": 300,
  "status": "WAITING_WA"
}
```

HTTP Status:

```text
200 OK
```

### Field Response

| Field | Tipe | Keterangan |
|---|---:|---|
| `success` | boolean | Status request. |
| `token` | string | OTP session token. Token ini digunakan untuk proses verify OTP. |
| `wa_link` | string | Link WhatsApp yang harus dibuka user untuk meminta kode OTP. |
| `text` | string | Isi pesan WhatsApp yang akan dikirim user. |
| `expires_in` | integer | Masa aktif session dalam detik. Default: 300 detik / 5 menit. |
| `status` | string | Status awal session, yaitu `WAITING_WA`. |

### Catatan Penting

- `token` pada response ini adalah **OTP Session Token**, bukan Bearer Access Token.
- Client wajib menyimpan `token` sementara untuk proses `/otp/verify`.
- Client menampilkan atau membuka `wa_link` agar user mengirim pesan WhatsApp ke GOWA.
- Session OTP aktif maksimal 5 menit sejak request dibuat.
- Jika user tidak mengirim WhatsApp sampai TTL habis, session otomatis expired.

---

## 6. Proses Inbound WhatsApp Verification

Bagian ini tidak dipanggil langsung oleh aplikasi client.

Setelah client menerima `wa_link`, user akan diarahkan ke WhatsApp dengan isi pesan:

```text
Permintaan Kode #OTP{token}
```

Contoh:

```text
Permintaan Kode #OTPFfjFwJIPhJYvCKMtLNA7YIe-9DJi9LvpgHhfsSa3q-c
```

Saat pesan tersebut diterima oleh GOWA, sistem akan:

1. Extract token dari pesan WhatsApp.
2. Cari session cache berdasarkan key `otp:req:{token}`.
3. Validasi session masih aktif.
4. Validasi status masih `WAITING_WA`.
5. Lock cache untuk mencegah duplicate processing.
6. Normalize nomor WhatsApp pengirim ke format `628xxx`.
7. Simpan nomor WhatsApp pengirim ke field `wa_number`.
8. Ubah status menjadi `OTP_SENT`.
9. Set `otp_expires_at` menjadi `now + 2 menit`.
10. Kirim kode OTP ke WhatsApp user.

### Balasan WhatsApp dari GOWA

```text
Kode OTP kamu: 309313
Berlaku 2 menit.
```

### Catatan Lifecycle

- TTL session tetap mengikuti waktu awal request, yaitu maksimal 5 menit.
- Saat inbound WhatsApp berhasil, TTL session tidak direset.
- `otp_expires_at` terpisah dari TTL session.
- OTP hanya bisa diverifikasi jika status sudah `OTP_SENT`.

---

## 7. Endpoint Verify OTP

Endpoint ini digunakan oleh aplikasi client untuk memverifikasi OTP yang diinput user.

```http
POST /otp/verify
```

Full URL:

```text
https://gowa.gresikkab.go.id/api/otp/verify
```

### Header

```http
Authorization: Bearer {ACCESS_TOKEN}
Accept: application/json
Content-Type: application/json
```

### Request Body

```json
{
  "token": "FfjFwJIPhJYvCKMtLNA7YIe-9DJi9LvpgHhfsSa3q-c",
  "otp": "309313"
}
```

### Field Request

| Field | Tipe | Wajib | Keterangan |
|---|---:|:---:|---|
| `token` | string | Ya | OTP Session Token dari response `/otp/request`. |
| `otp` | string | Ya | Kode OTP 6 digit yang diterima user melalui WhatsApp. |

### Contoh cURL

```bash
curl -X POST "https://gowa.gresikkab.go.id/api/otp/verify" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "token": "FfjFwJIPhJYvCKMtLNA7YIe-9DJi9LvpgHhfsSa3q-c",
    "otp": "309313"
  }'
```

### Success Response

```json
{
  "success": true,
  "status": "VERIFIED",
  "message": "OTP berhasil diverifikasi.",
  "data": {
    "app_id": "40118077-f9e5-4a42-82ee-af12aae2f2f5",
    "app_name": "inaproc test",
    "user_id": "1",
    "wa_number": "628xxxxxxxxxx",
    "verified_at": "2026-05-13 18:00:00"
  }
}
```

HTTP Status:

```text
200 OK
```

### Field Response Success

| Field | Tipe | Keterangan |
|---|---:|---|
| `success` | boolean | Bernilai `true` jika OTP valid. |
| `status` | string | Status final verifikasi, yaitu `VERIFIED`. |
| `message` | string | Pesan hasil verifikasi. |
| `data.app_id` | string | UUID aplikasi client. |
| `data.app_name` | string | Nama aplikasi client yang terdaftar di GOWA. |
| `data.user_id` | string | ID user dari aplikasi client. |
| `data.wa_number` | string | Nomor WhatsApp yang sudah terbukti memiliki akses ke session OTP. |
| `data.verified_at` | string | Waktu verifikasi berhasil. |

### Catatan Penting

Setelah OTP berhasil diverifikasi:

- Session cache langsung dihapus.
- Token tidak bisa digunakan ulang.
- Client boleh menganggap user sudah berhasil melakukan verifikasi nomor WhatsApp.

---

## 8. Contoh Error Response

### 8.1 App ID Tidak Ditemukan / Tidak Aktif

```json
{
  "success": false,
  "error": {
    "code": "APP_NOT_FOUND",
    "message": "ID App tidak ditemukan atau tidak aktif."
  },
  "meta": {}
}
```

HTTP Status:

```text
404 Not Found
```

---

### 8.2 OTP Belum Aktif

Terjadi jika client memanggil `/otp/verify` sebelum user mengirim pesan WhatsApp dari `wa_link`.

```json
{
  "success": false,
  "error": {
    "code": "OTP_NOT_READY",
    "message": "OTP belum aktif. Silakan kirim pesan WhatsApp dari link OTP terlebih dahulu."
  },
  "meta": {
    "status": "WAITING_WA"
  }
}
```

HTTP Status:

```text
409 Conflict
```

---

### 8.3 OTP Salah

```json
{
  "success": false,
  "error": {
    "code": "INVALID_OTP",
    "message": "Kode OTP salah."
  },
  "meta": {
    "remaining_attempt": 4
  }
}
```

HTTP Status:

```text
422 Unprocessable Entity
```

---

### 8.4 OTP Diblokir

Terjadi jika percobaan OTP sudah mencapai batas maksimal.

```json
{
  "success": false,
  "error": {
    "code": "OTP_BLOCKED",
    "message": "OTP diblokir karena percobaan sudah mencapai batas maksimal."
  },
  "meta": {
    "remaining_attempt": 0
  }
}
```

HTTP Status:

```text
429 Too Many Requests
```

---

### 8.5 Session OTP Expired

```json
{
  "success": false,
  "error": {
    "code": "SESSION_EXPIRED",
    "message": "Session OTP sudah kedaluwarsa. Silakan request OTP baru."
  },
  "meta": {}
}
```

HTTP Status:

```text
410 Gone
```

---

### 8.6 OTP Expired

```json
{
  "success": false,
  "error": {
    "code": "OTP_EXPIRED",
    "message": "Kode OTP sudah kedaluwarsa. Silakan request OTP baru."
  },
  "meta": {}
}
```

HTTP Status:

```text
410 Gone
```

---

### 8.7 Rate Limited

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "Terlalu banyak request OTP. Coba beberapa saat lagi."
  },
  "meta": {
    "retry_after": 45
  }
}
```

HTTP Status:

```text
429 Too Many Requests
```

---

## 9. Validasi dan Batasan

### Request OTP

| Validasi | Aturan |
|---|---|
| `app_id` | Wajib, string, UUID aplikasi yang terdaftar. |
| `user_id` | Wajib, string, identifier user dari client app. |
| Bearer Token | Wajib valid. |
| Rate limit | Dibatasi per user, aplikasi, dan IP. |

### Verify OTP

| Validasi | Aturan |
|---|---|
| `token` | Wajib, token dari response `/otp/request`. |
| `otp` | Wajib, 6 digit numeric. |
| Status session | Harus `OTP_SENT`. |
| Expiry OTP | Maksimal 2 menit sejak OTP dikirim ke WhatsApp. |
| Expiry session | Maksimal 5 menit sejak request OTP dibuat. |
| Attempt | Maksimal 5 kali percobaan. |

---

## 10. Konfigurasi Postman

Collection Postman dapat menggunakan collection-level Authorization.

### Collection Authorization

Type:

```text
Bearer Token
```

Token:

```text
{{access_token}}
```

> Disarankan menggunakan variable `access_token`, bukan `token`, agar tidak tertukar dengan OTP Session Token.

### Collection Variables

| Variable | Contoh Value | Keterangan |
|---|---|---|
| `base_url` | `https://gowa.gresikkab.go.id/api` | Base URL API GOWA. |
| `access_token` | `isi_bearer_access_token` | Token akses API dari GOWA. |
| `app_id` | `40118077-f9e5-4a42-82ee-af12aae2f2f5` | UUID aplikasi client. |
| `user_id` | `1` | ID user aplikasi client. |
| `otp_session_token` | kosong di awal | Diisi dari response `/otp/request`. |
| `otp` | kosong di awal | Diisi kode OTP yang diterima user dari WhatsApp. |

### Request 1: Request OTP

Method:

```text
POST
```

URL:

```text
{{base_url}}/otp/request
```

Body:

```json
{
  "app_id": "{{app_id}}",
  "user_id": "{{user_id}}"
}
```

Optional test script untuk menyimpan OTP session token otomatis:

```javascript
const json = pm.response.json();

if (json.success && json.token) {
  pm.collectionVariables.set("otp_session_token", json.token);
}
```

### Request 2: Verify OTP

Method:

```text
POST
```

URL:

```text
{{base_url}}/otp/verify
```

Body:

```json
{
  "token": "{{otp_session_token}}",
  "otp": "{{otp}}"
}
```

---

## 11. Hal yang Perlu Dilakukan Client App

Client app cukup melakukan:

1. Simpan `access_token` untuk Authorization header.
2. Kirim request ke `/otp/request`.
3. Simpan `token` dari response sebagai OTP Session Token.
4. Arahkan user ke `wa_link`.
5. Minta user input OTP yang diterima dari WhatsApp.
6. Kirim `token` dan `otp` ke `/otp/verify`.
7. Jika response `success = true`, user dianggap berhasil verifikasi WhatsApp.

Client app tidak perlu:

- Generate OTP.
- Menyimpan OTP.
- Membuat callback URL.
- Menerima callback dari GOWA.
- Mengirim nomor WhatsApp di request awal.
- Mengelola expiry OTP.
- Mengelola attempt limit.

---

## 12. Catatan Keamanan

- Gunakan HTTPS untuk semua request.
- Jangan kirim Bearer Access Token di query parameter.
- Simpan Bearer Access Token di server-side environment aplikasi client.
- Jangan expose Bearer Access Token ke frontend publik.
- Jangan log OTP secara plaintext di aplikasi client.
- OTP Session Token hanya berlaku sementara.
- OTP Session Token tidak boleh digunakan sebagai token autentikasi aplikasi.
- Setelah verify berhasil, GOWA menghapus session OTP.
- Jika attempt maksimal tercapai, GOWA menghapus session OTP.
- Jika session expired, client harus request OTP baru.

---

## 13. Ringkasan Endpoint

| Endpoint | Method | Auth | Fungsi |
|---|---|---|---|
| `/otp/request` | POST | Bearer Token | Membuat session OTP dan menghasilkan WhatsApp link. |
| `/otp/verify` | POST | Bearer Token | Memverifikasi OTP berdasarkan session token. |

---

## 14. Contoh Implementasi Sederhana Client

### Request OTP

```php
$response = Http::withToken(env('GOWA_ACCESS_TOKEN'))
    ->post('https://gowa.gresikkab.go.id/api/otp/request', [
        'app_id' => env('GOWA_APP_ID'),
        'user_id' => (string) $user->id,
    ]);

$data = $response->json();

if (!empty($data['success'])) {
    $otpSessionToken = $data['token'];
    $waLink = $data['wa_link'];

    // Simpan otpSessionToken sementara di session client app.
    session(['otp_session_token' => $otpSessionToken]);

    // Redirect atau tampilkan link WhatsApp ke user.
}
```

### Verify OTP

```php
$response = Http::withToken(env('GOWA_ACCESS_TOKEN'))
    ->post('https://gowa.gresikkab.go.id/api/otp/verify', [
        'token' => session('otp_session_token'),
        'otp' => $request->input('otp'),
    ]);

$data = $response->json();

if (!empty($data['success'])) {
    // OTP valid.
    // Nomor WhatsApp user tersedia di:
    $waNumber = $data['data']['wa_number'];

    // Lanjutkan proses verifikasi di aplikasi client.
}
```

