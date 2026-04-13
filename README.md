# PDF Service — Gotenberg

Service untuk generate PDF dari HTML/URL dan merge PDF/gambar menggunakan **Gotenberg** langsung (tanpa Traefik).

---

## Stack

| Komponen | Fungsi |
|---|---|
| Gotenberg 8 | PDF engine (Chromium + LibreOffice) |

---

## Fitur

- HTML → PDF (simple & complex dengan JS)
- URL → PDF
- Merge PDF
- Image → PDF
- Basic Auth di Gotenberg (`GOTENBERG_USER` / `GOTENBERG_PASS`)
- Credentials via `.env`

---

## Struktur Folder

```
project/
├── docker-compose.yml
├── .env                  ← buat dari .env.example
└── README.md
```

---

## Setup

### 1. Copy `.env`

```bash
cp .env.example .env
```

Edit `.env` sesuai kebutuhan:

```env
GOTENBERG_USER=admin
GOTENBERG_PASS=your_strong_password
```

### 2. Jalankan

```bash
docker compose up -d
```

### 3. Cek Status (dari dalam container)

```bash
docker compose exec gotenberg \
  curl -u admin:your_password http://localhost:3000/health
```

Response sukses:
```json
{"status":"up","details":{"chromium":{"status":"up"},"libreoffice":{"status":"up"}}}
```

---

## Testing

Cek **hanya status HTTP** (tanpa menyimpan body) pakai `-w "%{http_code}"`:

### Basic auth aktif

Tanpa `-u` → harus **`401`**:

```bash
docker compose exec gotenberg \
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/health
```

Dengan kredensial dari `.env` → harus **`200`**:

```bash
docker compose exec gotenberg \
  curl -s -o /dev/null -w "%{http_code}\n" \
    -u admin:your_password \
    http://localhost:3000/health
```

### POST URL → PDF

Tanpa auth → **`401`**:

```bash
docker compose exec gotenberg \
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST http://localhost:3000/forms/chromium/convert/url \
    -F "url=https://example.com"
```

Dengan auth → **`200`** jika konversi berhasil (bisa **`4xx`** kalau situs / Chromium menolak):

```bash
docker compose exec gotenberg \
  curl -s -o /dev/null -w "%{http_code}\n" \
    -u admin:your_password \
    -X POST http://localhost:3000/forms/chromium/convert/url \
    -F "url=https://example.com"
```

Kalau pakai `-o output.pdf` **tanpa** `-u` tapi file hasilnya nyaris kosong (cuma puluhan byte), itu biasanya body error (mis. JSON), bukan PDF — tambahkan `-u admin:your_password`.

---

## Penggunaan API

### Health Check

```bash
docker compose exec gotenberg \
  curl -u admin:password http://localhost:3000/health
```

### HTML → PDF

```bash
docker compose exec gotenberg sh -lc 'curl -u admin:password \
  -X POST http://localhost:3000/forms/chromium/convert/html \
  -F "files=@/tmp/index.html" \
  -o /tmp/output.pdf'
```

### URL → PDF

```bash
docker compose exec gotenberg \
  curl -u admin:password \
    -X POST http://localhost:3000/forms/chromium/convert/url \
    -F "url=https://example.com" \
    -o /tmp/output.pdf
```

### Merge PDF

```bash
docker compose exec gotenberg sh -lc 'curl -u admin:password \
  -X POST http://localhost:3000/forms/pdfengines/merge \
  -F "files=@/tmp/file1.pdf" \
  -F "files=@/tmp/file2.pdf" \
  -o /tmp/merged.pdf'
```

### Image → PDF

```bash
docker compose exec gotenberg sh -lc 'curl -u admin:password \
  -X POST http://localhost:3000/forms/chromium/convert/html \
  -F "files=@/tmp/index.html" \
  -o /tmp/output.pdf'
```

> Wrap gambar dalam HTML sederhana, lalu convert via Chromium.

---

## Konfigurasi

Basic auth API aktif dari `docker-compose.yml` dengan:
- `GOTENBERG_API_ENABLE_BASIC_AUTH=true`
- `GOTENBERG_API_BASIC_AUTH_USERNAME=${GOTENBERG_USER}`
- `GOTENBERG_API_BASIC_AUTH_PASSWORD=${GOTENBERG_PASS}`

### Resource Limit Gotenberg

| Setting | Default |
|---|---|
| Memory limit | 1G |
| CPU limit | 1.0 core |
| Memory reservation | 512M |
| Chromium restart after | 50 req |
| LibreOffice restart after | 10 req |
| API timeout | 30s |

---

## Tips Production

1. Ganti `GOTENBERG_USER` / `GOTENBERG_PASS` default sebelum deploy
2. Simpan `.env` di secret manager, jangan commit ke git
3. Tambahkan `.env` ke `.gitignore`
4. Untuk akses dari host, publish port `3000` di service `gotenberg` bila diperlukan
5. Tambahkan reverse proxy/WAF hanya jika butuh rate limit atau proteksi edge

---

## Dokumentasi Lengkap

- [Gotenberg Docs](https://gotenberg.dev/docs)
