# Dokumentasi Akses API GOWA Number Verification V1

Dokumentasi ini menjelaskan cara aplikasi client menggunakan **GOWA Number Verification V1** untuk memverifikasi nomor WhatsApp melalui pesan inbound WhatsApp.

Berbeda dengan **Managed WhatsApp OTP**, Number Verification tidak menggunakan kode OTP dan tidak memiliki endpoint `/verify`. Hasil verifikasi dikirim oleh GOWA ke `callback_url` aplikasi client.

> **Penting:** GOWA tidak mengirim pesan WhatsApp balasan pada flow Number Verification. User hanya mengirim pesan verifikasi ke nomor WhatsApp GOWA.

---

# 1. Overview

## 1.1 Tujuan Layanan

Number Verification digunakan untuk membuktikan bahwa suatu nomor WhatsApp memiliki akses ke transaksi verifikasi yang dibuat oleh aplikasi client.

Alur sederhananya:

```text
Client membuat request
        ↓
GOWA memberikan wa_link
        ↓
User membuka WhatsApp
        ↓
User mengirim token verifikasi
        ↓
GOWA memverifikasi sender
        ↓
GOWA mengirim callback ke client
```

Nomor WhatsApp yang pertama kali mengirim token valid sebelum masa verifikasi habis menjadi `wa_number` yang terverifikasi.

## 1.2 Perbedaan dengan Managed OTP

| Fitur | Number Verification | Managed OTP |
|---|---|---|
| User mengirim WhatsApp ke GOWA | Ya | Ya |
| GOWA mengirim balasan WhatsApp | Tidak | Ya, mengirim OTP |
| User memasukkan OTP | Tidak | Ya |
| Endpoint verify | Tidak ada | `/otp/verify` |
| Hasil verifikasi | Callback ke client | Response `/otp/verify` |
| Callback URL | Wajib | Tidak digunakan |
| Nomor terverifikasi | Sender pertama token valid | Sender session OTP |

Number Verification cocok jika aplikasi hanya membutuhkan hasil kepemilikan akses terhadap nomor WhatsApp tanpa proses input OTP.

## 1.3 Base URL

```text
https://gowa.gresikkab.go.id/api
```

Endpoint:

```http
POST /wa-verification/request
```

Full URL:

```text
https://gowa.gresikkab.go.id/api/wa-verification/request
```

---

# 2. Prerequisites

Sebelum menggunakan Number Verification, client harus memiliki:

- **Bearer Access Token** GOWA.
- **Verification App** yang sudah terdaftar di GOWA.
- `app_id` dari Verification App.
- `callback_url` aktif untuk menerima hasil verifikasi.

Contoh:

```text
Access Token : {ACCESS_TOKEN}
App ID       : 40118077-f9e5-4a42-82ee-af12aae2f2f5
Callback URL : https://client.example.com/webhook/gowa-verification
```

`callback_url` disimpan pada konfigurasi Verification App di GOWA. Client tidak perlu mengirim `callback_url` pada setiap request.

---

# 3. Authentication

Endpoint Number Verification wajib menggunakan **Bearer Token**.

Bearer Token adalah **Access Token untuk mengakses API GOWA**, bukan verification token yang dikirim melalui WhatsApp.

## 3.1 Header Wajib

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

Jika Bearer Token tidak valid, request akan ditolak oleh middleware autentikasi GOWA.

Umumnya:

```text
401 Unauthorized
```

Token yang tidak aktif atau tidak diizinkan dapat menghasilkan:

```text
403 Forbidden
```

> **Penting:** Bearer Access Token digunakan untuk autentikasi API. Verification token hanya digunakan pada pesan WhatsApp `#GOWA-v1:{token}`.

---

# 4. Verification Flow

## 4.1 Flow Utama

```text
Client Backend
      |
      | POST /api/wa-verification/request
      | Authorization: Bearer ...
      | app_id
      | client_reference_id (optional)
      | content (optional)
      v
    GOWA
      |
      | Validasi app
      | Generate verification_id
      | Generate verification token
      | Simpan transaksi WAITING_WA
      | Return wa_link
      v
Client App
      |
      | Tampilkan / buka wa_link
      v
User WhatsApp
      |
      | Kirim:
      | #GOWA-v1:{token}
      v
GOWA Inbound WhatsApp
      |
      | Validasi token
      | Validasi expiry
      | First valid sender wins
      |
      +------------------+
      |                  |
      v                  v
  VERIFIED            EXPIRED
      |                  |
      +--------+---------+
               |
               | HTTP Callback
               v
          Client Backend
               |
               | Return HTTP 2xx
               v
            Selesai
```

## 4.2 Proses WhatsApp

Setelah request berhasil, client mengarahkan user ke:

```text
data.wa_link
```

WhatsApp akan terbuka dengan pesan:

```text
#GOWA-v1:{token}
```

Contoh:

```text
#GOWA-v1:OMwvqg3qgV73Fj0BJaUoYd8zD0f6PDiR7yJuPAPdkW0
```

Pesan harus dikirim tanpa perubahan.

Ketika pesan diterima, GOWA:

1. Membaca nomor WhatsApp pengirim.
2. Memvalidasi format protocol.
3. Mencocokkan token dengan transaksi verifikasi.
4. Memastikan transaksi masih `WAITING_WA`.
5. Memastikan Verification App masih aktif.
6. Membandingkan waktu inbound dengan waktu expiry.
7. Menetapkan hasil `VERIFIED` atau `EXPIRED`.
8. Membuat callback event jika diperlukan.
9. Mengirim callback ke aplikasi client.

> **Penting:** Tidak ada pesan WhatsApp outbound dari GOWA pada flow Number Verification, termasuk saat verification berhasil, expired, token salah, format salah, duplicate, app tidak aktif, atau callback gagal.

---

# 5. API Reference

## 5.1 Membuat Transaksi Verifikasi

```http
POST /wa-verification/request
```

Full URL:

```text
https://gowa.gresikkab.go.id/api/wa-verification/request
```

### Header

```http
Authorization: Bearer {ACCESS_TOKEN}
Accept: application/json
Content-Type: application/json
```

### Minimal Request

```json
{
  "app_id": "40118077-f9e5-4a42-82ee-af12aae2f2f5"
}
```

### Recommended Request

```json
{
  "app_id": "40118077-f9e5-4a42-82ee-af12aae2f2f5",
  "client_reference_id": "login-user-123",
  "content": {
    "user_id": "123",
    "purpose": "login"
  }
}
```

### Field Request

| Field | Tipe | Wajib | Keterangan |
|---|---|:---:|---|
| `app_id` | string | Ya | UUID Verification App yang terdaftar dan aktif di GOWA. |
| `client_reference_id` | string / null | Tidak | Correlation ID milik client. Maksimal 255 karakter. |
| `content` | JSON value | Tidak | Payload bebas milik client. Maksimal 32 KB. |

## 5.2 `client_reference_id` sebagai Correlation ID

`client_reference_id` membantu client menghubungkan callback dengan transaksi atau data internal.

Contoh:

```json
{
  "client_reference_id": "login-user-123"
}
```

Karakteristik:

- optional;
- nullable;
- maksimal 255 karakter;
- tidak harus unique;
- bukan credential;
- bukan verification token;
- bukan idempotency key.

Contoh nilai:

```text
user-123
login-session-abc
reset-password-0193
transaction-928181
```

Dua transaksi boleh menggunakan `client_reference_id` yang sama.

## 5.3 Membawa Data Client melalui `content`

`content` digunakan untuk membawa konteks milik aplikasi client.

Contoh object:

```json
{
  "content": {
    "user_id": "123",
    "purpose": "login",
    "session_id": "abc-123"
  }
}
```

Contoh string opaque:

```json
{
  "content": "ENCRYPTED_OR_OPAQUE_CLIENT_VALUE"
}
```

Tipe yang didukung:

```text
object
array
string
number
boolean
null
```

Batas maksimum:

```text
32768 bytes / 32 KB
```

GOWA tidak menggunakan isi `content` untuk menentukan hasil verifikasi.

> **Catatan:** Untuk V1, sebaiknya jangan menggunakan perbedaan antara empty object `{}` dan empty array `[]` sebagai sinyal logika bisnis.

## 5.4 Contoh cURL

```bash
curl -X POST "https://gowa.gresikkab.go.id/api/wa-verification/request" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "40118077-f9e5-4a42-82ee-af12aae2f2f5",
    "client_reference_id": "login-user-123",
    "content": {
      "user_id": "123",
      "purpose": "login"
    }
  }'
```

## 5.5 Success Response

HTTP Status:

```text
200 OK
```

```json
{
  "success": true,
  "data": {
    "verification_id": "0f75cdca-5292-4ff7-8f73-c342ae8097f2",
    "token": "OMwvqg3qgV73Fj0BJaUoYd8zD0f6PDiR7yJuPAPdkW0",
    "message": "#GOWA-v1:OMwvqg3qgV73Fj0BJaUoYd8zD0f6PDiR7yJuPAPdkW0",
    "wa_link": "https://wa.me/6281217889986?text=%23GOWA-v1%3AOMwvqg3qgV73Fj0BJaUoYd8zD0f6PDiR7yJuPAPdkW0",
    "expires_in": 300,
    "status": "WAITING_WA"
  }
}
```

### Field Response

| Field | Tipe | Keterangan |
|---|---|---|
| `success` | boolean | Bernilai `true` jika transaksi berhasil dibuat. |
| `data.verification_id` | UUID string | ID transaksi verifikasi. Sebaiknya disimpan client. |
| `data.token` | string | One-time verification token. |
| `data.message` | string | Pesan yang harus dikirim melalui WhatsApp tanpa perubahan. |
| `data.wa_link` | string | WhatsApp deep link yang dapat dibuka user. |
| `data.expires_in` | integer | Masa aktif verifikasi dalam detik. Saat ini 300 detik. |
| `data.status` | string | Status awal transaksi, yaitu `WAITING_WA`. |

Minimal data yang sebaiknya disimpan client:

```text
verification_id
client_reference_id
status lokal
expires_at / waktu timeout UI
```

## 5.6 Ringkasan Error Response

| HTTP Status | Error Code | Kondisi |
|---|---|---|
| `400` | `INVALID_REQUEST` | Request body bukan JSON object yang valid. |
| `404` | `APP_NOT_FOUND` | App tidak ditemukan atau tidak aktif. |
| `409` | `CALLBACK_NOT_CONFIGURED` | Callback URL belum tersedia atau tidak valid. |
| `413` | `CONTENT_TOO_LARGE` | `content` melebihi 32 KB. |
| `422` | `INVALID_REQUEST` | Field request tidak valid. |
| `429` | `RATE_LIMITED` | Rate limit terlampaui. |
| `503` | `SERVICE_UNAVAILABLE` | Service atau storage tidak tersedia. |

### Invalid Request

```json
{
  "success": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Request body tidak valid."
  }
}
```

### App Tidak Ditemukan atau Tidak Aktif

```json
{
  "success": false,
  "error": {
    "code": "APP_NOT_FOUND",
    "message": "App ID tidak ditemukan atau tidak aktif."
  }
}
```

### Callback Belum Dikonfigurasi

```json
{
  "success": false,
  "error": {
    "code": "CALLBACK_NOT_CONFIGURED",
    "message": "Callback URL aplikasi belum dikonfigurasi."
  }
}
```

### `client_reference_id` Tidak Valid

```json
{
  "success": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "client_reference_id tidak valid."
  }
}
```

### Content Terlalu Besar

```json
{
  "success": false,
  "error": {
    "code": "CONTENT_TOO_LARGE",
    "message": "Content melebihi batas maksimum 32 KB."
  }
}
```

### Rate Limited

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "Terlalu banyak permintaan. Silakan coba kembali.",
    "retry_after": 45
  }
}
```

### Service Unavailable

```json
{
  "success": false,
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Number verification service sementara tidak tersedia."
  }
}
```

---

# 6. Verification Lifecycle

## 6.1 Status Transaksi

| Status | Keterangan |
|---|---|
| `WAITING_WA` | Transaksi sudah dibuat dan menunggu pesan WhatsApp user. |
| `VERIFIED` | Token valid diterima sebelum expiry. |
| `EXPIRED` | Token dikenal tetapi diterima saat atau setelah expiry. |
| `APP_UNAVAILABLE` | Verification App tidak tersedia atau tidak aktif saat inbound diproses. |

Terminal state:

```text
VERIFIED
EXPIRED
APP_UNAVAILABLE
```

Setelah menjadi terminal, status tidak berubah lagi.

```text
VERIFIED        -X-> EXPIRED
EXPIRED         -X-> VERIFIED
APP_UNAVAILABLE -X-> VERIFIED
```

## 6.2 First Sender Wins

Number Verification V1 menggunakan aturan:

```text
first valid sender wins
```

Nomor WhatsApp pertama yang mengirim token valid sebelum expiry menjadi `wa_number`.

Contoh:

```text
Token XYZ

Nomor A mengirim lebih dahulu:
628111111111

Hasil:
status    = VERIFIED
wa_number = 628111111111
```

Jika Nomor B kemudian mengirim token yang sama, `wa_number` tetap milik Nomor A dan tidak ada callback kedua.

### Makna Verifikasi

Number Verification membuktikan bahwa `wa_number` mempunyai akses ke transaksi verifikasi dan berhasil mengirim token valid lebih dahulu.

Number Verification **tidak membuktikan** bahwa nomor tersebut sama dengan nomor yang sebelumnya disimpan atau diinput di aplikasi client.

Jika client memiliki expected phone number, pengecekan kecocokan nomor tetap menjadi logika bisnis client.

## 6.3 Expiration dan Retention

Masa aktif verifikasi:

```text
5 menit
```

Retention record:

```text
24 jam
```

Keduanya berbeda:

```text
Verification validity : 5 menit
Record retention      : 24 jam
```

Retention memungkinkan GOWA mengenali late inbound, duplicate inbound, dan melakukan callback retry setelah masa verifikasi selesai.

Setelah retention berakhir dan record dibersihkan, token lama tidak lagi menghasilkan callback.

## 6.4 Tidak Ada Automatic Expired Callback

Jika user tidak pernah mengirim WhatsApp sampai waktu verifikasi habis, GOWA **tidak mengirim callback expired secara otomatis**.

Client sudah menerima:

```json
{
  "expires_in": 300
}
```

Gunakan nilai tersebut untuk menentukan timeout pada UI.

Contoh:

```text
00:00  request dibuat
05:00  UI client menampilkan bahwa waktu verifikasi telah habis
```

Callback `number_verification.expired` hanya muncul jika user benar-benar mengirim token setelah expiry dan transaction masih tersedia.

> **Penting:** Jangan menunggu callback `EXPIRED` untuk menutup UI timeout.

## 6.5 Tidak Ada Request Idempotency

V1 tidak menggunakan:

```text
Idempotency-Key
```

Dua request create menghasilkan dua transaksi independen:

```text
Request 1 -> transaction A -> token A
Request 2 -> transaction B -> token B
```

Jika request diulang setelah network timeout, request baru dapat menghasilkan transaction baru.

---

# 7. Callback

## 7.1 Callback Endpoint

Client harus menyediakan endpoint HTTP yang dapat menerima:

```http
POST {CALLBACK_URL}
Content-Type: application/json
X-GOWA-Event-Id: {event_id}
X-GOWA-Verification-Id: {verification_id}
```

Contoh:

```text
https://client.example.com/webhook/gowa-verification
```

Client harus mengembalikan HTTP `2xx` jika callback berhasil diterima.

Contoh:

```json
{
  "success": true
}
```

Body response tidak memiliki format khusus. HTTP status `2xx` adalah indikator utama bahwa callback berhasil diterima.

> **Penting:** Callback sebaiknya merespons secepat mungkin. Proses bisnis yang berat dapat diteruskan secara asynchronous di sisi client.

## 7.2 Verified Event

Event:

```text
number_verification.verified
```

Contoh payload:

```json
{
  "event": "number_verification.verified",
  "event_id": "7cd57511-379e-4c95-a0ea-6ef6daa44919",
  "verification_id": "0f75cdca-5292-4ff7-8f73-c342ae8097f2",
  "app_id": "40118077-f9e5-4a42-82ee-af12aae2f2f5",
  "client_reference_id": "login-user-123",
  "status": "VERIFIED",
  "wa_number": "628123456789",
  "content": {
    "user_id": "123",
    "purpose": "login"
  },
  "occurred_at": "2026-10-13T10:30:00+07:00"
}
```

| Field | Keterangan |
|---|---|
| `event` | Nama event. |
| `event_id` | UUID event callback untuk deduplication. |
| `verification_id` | ID transaksi yang dibuat sebelumnya. |
| `app_id` | UUID Verification App. |
| `client_reference_id` | Correlation ID milik client. |
| `status` | `VERIFIED`. |
| `wa_number` | Nomor WhatsApp yang berhasil diverifikasi. |
| `content` | Payload dari request awal. |
| `occurred_at` | Waktu event dalam format ISO 8601. |

## 7.3 Expired Event

Event:

```text
number_verification.expired
```

Callback expired hanya dikirim jika user mengirim verification token setelah masa verifikasi habis. Transaction tersebut juga harus masih tersedia di GOWA.

Contoh:

```json
{
  "event": "number_verification.expired",
  "event_id": "8397e30c-a65b-4eb3-b81e-67807e8db11a",
  "verification_id": "0f75cdca-5292-4ff7-8f73-c342ae8097f2",
  "app_id": "40118077-f9e5-4a42-82ee-af12aae2f2f5",
  "client_reference_id": "login-user-123",
  "status": "EXPIRED",
  "wa_number": "628123456789",
  "content": {
    "user_id": "123",
    "purpose": "login"
  },
  "error": {
    "code": "VERIFICATION_EXPIRED",
    "message": "Verification request has expired."
  },
  "occurred_at": "2026-10-13T10:36:00+07:00"
}
```

`wa_number` tetap tersedia karena GOWA mengetahui sender yang melakukan late attempt.

## 7.4 Callback Retry Policy

Callback pertama dikirim segera setelah status menjadi `VERIFIED` atau `EXPIRED`.

Maximum:

```text
3 attempts
```

Schedule:

```text
Attempt 1 : immediate
Attempt 2 : +1 menit
Attempt 3 : +5 menit
```

Timeout:

```text
Connect timeout : 1 detik
Total timeout   : 5 detik
```

Retry dilakukan untuk:

```text
connection failure
DNS / network failure
timeout
HTTP 408
HTTP 425
HTTP 429
HTTP 5xx
```

Tidak ada scheduled retry untuk:

```text
HTTP 3xx
HTTP 400
HTTP 401
HTTP 403
HTTP 404
HTTP 405
HTTP 410
HTTP 422
non-retryable HTTP 4xx lainnya
```

Jika attempt ketiga masih gagal, callback tidak di-retry otomatis lagi.

> **Penting:** Kegagalan callback tidak membatalkan hasil verification. Jika status sudah `VERIFIED`, status tetap `VERIFIED`.

## 7.5 Idempotency Callback dengan `event_id`

Semua retry untuk event yang sama menggunakan `event_id` yang sama.

```text
Attempt 1 -> event_id = ABC
Attempt 2 -> event_id = ABC
Attempt 3 -> event_id = ABC
```

Client harus melakukan deduplication berdasarkan `event_id`.

Pseudo flow:

```text
Receive callback
      |
      v
event_id sudah diproses?
      |
   +--+--+
   |     |
  Ya    Tidak
   |     |
return   proses event
2xx      simpan event_id
         return 2xx
```

Ini penting karena client mungkin sudah memproses callback, tetapi response `2xx` tidak sampai ke GOWA akibat timeout atau gangguan jaringan.

## 7.6 Callback Retry Menggunakan URL Terbaru

Setiap attempt menggunakan `callback_url` terbaru dari Verification App.

Contoh:

```text
Attempt 1
https://old.example.com/callback
-> gagal

Callback URL diubah menjadi:
https://new.example.com/callback

Attempt 2
https://new.example.com/callback
```

Callback URL tidak di-snapshot saat transaksi dibuat.

## 7.7 Kondisi yang Tidak Menghasilkan Callback

GOWA hanya mempunyai dua business callback event:

```text
number_verification.verified
number_verification.expired
```

Tidak ada callback untuk:

```text
invalid API request
invalid Bearer Token
INVALID_FORMAT
UNKNOWN_TOKEN
DUPLICATE
APP_UNAVAILABLE
rate-limited inbound
```

Jika Verification App menjadi inactive atau dihapus setelah transaksi dibuat tetapi sebelum inbound diproses, transaksi dapat menjadi `APP_UNAVAILABLE` tanpa callback.

---

# 8. Limits and Security

## 8.1 Rate Limit

| Scope | Limit |
|---|---:|
| Request berdasarkan source IP | 120 request / menit |
| Request berdasarkan `app_id + source IP` | 60 request / menit |
| Inbound berdasarkan nomor WhatsApp sender | 10 pesan / menit |

Tidak ada limiter khusus berdasarkan Bearer Token pada Number Verification V1.

Jika API request terkena rate limit:

```text
HTTP 429 Too Many Requests
```

Response membawa `retry_after` dalam detik.

## 8.2 Batas Data

| Item | Batas |
|---|---|
| Verification TTL | 300 detik / 5 menit |
| Record retention | 24 jam |
| `client_reference_id` | 255 karakter |
| `content` | 32768 bytes / 32 KB |
| Callback attempts | 3 |
| Callback connect timeout | 1 detik |
| Callback total timeout | 5 detik |

## 8.3 Keamanan API

Disarankan:

- gunakan HTTPS;
- simpan Bearer Access Token hanya di backend/server environment;
- jangan expose Bearer Access Token ke frontend publik;
- jangan kirim Bearer Token melalui query parameter;
- simpan `verification_id` untuk menghubungkan callback dengan transaksi yang dibuat.

## 8.4 Keamanan Callback V1

V1 belum menggunakan:

```text
HMAC
App Secret
Public/Private Key signature
X-Signature
```

Karena itu client disarankan:

1. Gunakan callback URL HTTPS.
2. Pastikan `verification_id` dikenal oleh sistem client.
3. Cocokkan `app_id` dengan aplikasi yang digunakan.
4. Gunakan `event_id` untuk deduplication.
5. Jangan memperlakukan `client_reference_id` sebagai credential.
6. Jangan memperlakukan `content` sebagai bukti autentikasi dari GOWA.

`wa_number` merupakan hasil utama Number Verification.

---

# 9. Integration Examples

## 9.1 Konfigurasi Postman

### Collection Authorization

Type:

```text
Bearer Token
```

Token:

```text
{{access_token}}
```

### Collection Variables

| Variable | Contoh Value | Keterangan |
|---|---|---|
| `base_url` | `https://gowa.gresikkab.go.id/api` | Base URL GOWA. |
| `access_token` | `isi_bearer_access_token` | Bearer Access Token. |
| `app_id` | `40118077-f9e5-4a42-82ee-af12aae2f2f5` | UUID Verification App. |
| `client_reference_id` | `login-user-123` | Correlation ID client. |
| `verification_id` | kosong di awal | Diambil dari response request. |
| `wa_link` | kosong di awal | Diambil dari response request. |

### Request

Method:

```text
POST
```

URL:

```text
{{base_url}}/wa-verification/request
```

Body:

```json
{
  "app_id": "{{app_id}}",
  "client_reference_id": "{{client_reference_id}}",
  "content": {
    "purpose": "login"
  }
}
```

Optional test script:

```javascript
const json = pm.response.json();

if (json.success && json.data) {
    pm.collectionVariables.set(
        "verification_id",
        json.data.verification_id
    );

    pm.collectionVariables.set(
        "wa_link",
        json.data.wa_link
    );
}
```

Setelah request berhasil, buka `wa_link` dan kirim pesan melalui WhatsApp.

Callback dikirim ke `callback_url` Verification App, bukan ke request Postman tersebut.

## 9.2 Contoh Request dari Laravel/PHP

```php
use Illuminate\Support\Facades\Http;

$response = Http::withToken(env('GOWA_ACCESS_TOKEN'))
    ->acceptJson()
    ->post(
        'https://gowa.gresikkab.go.id/api/wa-verification/request',
        [
            'app_id' => env('GOWA_VERIFICATION_APP_ID'),
            'client_reference_id' => 'login-user-' . $user->id,
            'content' => [
                'user_id' => (string) $user->id,
                'purpose' => 'login',
            ],
        ]
    );

$data = $response->json();

if ($response->successful() && !empty($data['success'])) {
    $verificationId = $data['data']['verification_id'];
    $waLink = $data['data']['wa_link'];
    $expiresIn = $data['data']['expires_in'];

    // Simpan transaction client.
    // status = WAITING_WA
    // expires_at = now() + $expiresIn

    // Kirim $waLink ke frontend.
}
```

## 9.3 Contoh Callback Handler Laravel/PHP

Contoh route:

```php
Route::post(
    '/webhook/gowa-verification',
    [GowaVerificationController::class, 'callback']
);
```

Contoh controller:

```php
use Illuminate\Http\Request;

public function callback(Request $request)
{
    $eventId = $request->header('X-GOWA-Event-Id');
    $verificationId = $request->header('X-GOWA-Verification-Id');

    $payload = $request->all();

    $verification = Verification::where(
        'verification_id',
        $verificationId
    )->first();

    if (!$verification) {
        // Sesuaikan policy aplikasi client.
        return response()->json(['success' => true]);
    }

    if (VerificationEvent::where('event_id', $eventId)->exists()) {
        return response()->json(['success' => true]);
    }

    if (($payload['event'] ?? null) === 'number_verification.verified') {
        $verification->update([
            'status' => 'VERIFIED',
            'wa_number' => $payload['wa_number'] ?? null,
            'verified_at' => $payload['occurred_at'] ?? now(),
        ]);
    }

    if (($payload['event'] ?? null) === 'number_verification.expired') {
        $verification->update([
            'status' => 'EXPIRED',
        ]);
    }

    VerificationEvent::create([
        'event_id' => $eventId,
        'verification_id' => $verificationId,
    ]);

    return response()->json([
        'success' => true,
    ]);
}
```

Nama table, model, dan struktur database pada aplikasi client bebas disesuaikan.

## 9.4 Flow Frontend yang Disarankan

```text
User menekan "Verifikasi WhatsApp"
                |
                v
Frontend meminta backend membuat verification
                |
                v
Backend -> GOWA /wa-verification/request
                |
                v
Frontend menerima wa_link
                |
                v
Buka WhatsApp
                |
                v
User mengirim pesan
                |
                v
Frontend menampilkan:
"Menunggu verifikasi..."
                |
                v
Callback GOWA masuk ke backend client
                |
                v
Backend mengubah status transaction
                |
                v
Frontend polling / websocket / refresh
                |
                v
VERIFIED
```

Callback dikirim ke backend client, bukan langsung ke browser/frontend.

Untuk timeout UI, gunakan:

```text
expires_in = 300
```

Contoh sederhana:

```javascript
const expiresIn = response.data.expires_in;

setTimeout(() => {
    // Tampilkan:
    // "Waktu verifikasi telah habis. Silakan coba kembali."
}, expiresIn * 1000);
```

---

# 10. Integration Checklist and Quick Reference

## 10.1 Integration Checklist

Sebelum production:

```text
[ ] Bearer Access Token tersedia
[ ] Verification App aktif
[ ] app_id sudah benar
[ ] callback_url menggunakan HTTPS
[ ] callback_url dapat menerima POST JSON
[ ] callback mengembalikan HTTP 2xx
[ ] verification_id disimpan client
[ ] client_reference_id digunakan jika diperlukan
[ ] wa_link berhasil dibuka
[ ] user berhasil mengirim #GOWA-v1:{token}
[ ] verified callback berhasil diterima
[ ] wa_number berhasil disimpan client
[ ] event_id dideduplicate
[ ] UI menggunakan expires_in untuk timeout
[ ] expired callback dapat diproses
[ ] retry callback tidak menghasilkan duplicate business action
[ ] expected phone number divalidasi client jika diperlukan
```

## 10.2 Quick Reference

### Endpoint

| Endpoint | Method | Auth | Fungsi |
|---|---|---|---|
| `/wa-verification/request` | POST | Bearer Token | Membuat transaksi verifikasi dan menghasilkan `wa_link`. |

Tidak ada endpoint `/verify`.

### Event

```text
number_verification.verified
number_verification.expired
```

### Status

```text
WAITING_WA
VERIFIED
EXPIRED
APP_UNAVAILABLE
```

### Flow Singkat

```text
1. POST /wa-verification/request
2. Simpan verification_id
3. Buka wa_link
4. User kirim pesan WhatsApp
5. Tunggu callback
6. VERIFIED -> simpan wa_number
7. Return HTTP 2xx
```

### Nilai Penting

```text
Verification TTL : 5 menit
Retention        : 24 jam
Callback retry   : 3 attempts
Retry schedule   : immediate, +1 menit, +5 menit
Content limit    : 32 KB
Request limit    : 120/min per IP
App + IP limit   : 60/min
Inbound WA limit : 10/min per sender
```

### Kebutuhan Client

```text
Bearer Access Token
+
app_id
+
callback_url
+
endpoint create verification
+
callback handler
```

Tanpa OTP, tanpa endpoint verify, dan tanpa WhatsApp outbound dari GOWA.
