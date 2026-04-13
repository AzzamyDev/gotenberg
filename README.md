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

### 3. Cek Status

```bash
curl http://localhost:7100/health
```

Response sukses:
```json
{"status":"up","details":{"chromium":{"status":"up"},"libreoffice":{"status":"up"}}}
```

---

## Testing

Cek **hanya status HTTP** (tanpa menyimpan body) pakai `-w "%{http_code}"`:

### Basic auth aktif

Catatan: endpoint `/health` bersifat publik untuk healthcheck, jadi bisa tetap **`200`** tanpa auth.

Validasi auth harus pakai endpoint `/forms/*`.

Tanpa `-u` pada endpoint konversi → harus **`401`**:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -X POST http://localhost:7100/forms/chromium/convert/url \
  -F "url=https://google.com"
```

Dengan kredensial dari `.env` pada endpoint konversi → harus **`200`** jika konversi berhasil:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -u admin:your_password \
  -X POST http://localhost:7100/forms/chromium/convert/url \
  -F "url=https://google.com"
```

### POST URL → PDF

```bash
curl -u admin:your_password \
  -X POST http://localhost:7100/forms/chromium/convert/url \
  -F "url=https://google.com" \
  -o output.pdf
```

Kalau pakai `-o output.pdf` **tanpa** `-u` tapi file hasilnya nyaris kosong (cuma puluhan byte), itu biasanya body error (mis. JSON), bukan PDF — tambahkan `-u admin:your_password`.

---

## Penggunaan API

### Health Check

```bash
curl -u admin:password http://localhost:7100/health
```

### HTML → PDF

```bash
curl -u admin:password \
  -X POST http://localhost:7100/forms/chromium/convert/html \
  -F "files=@index.html" \
  -o output.pdf
```

### URL → PDF

```bash
curl -u admin:password \
  -X POST http://localhost:7100/forms/chromium/convert/url \
  -F "url=https://google.com" \
  -o output.pdf
```

### Merge PDF

```bash
curl -u admin:password \
  -X POST http://localhost:7100/forms/pdfengines/merge \
  -F "files=@file1.pdf" \
  -F "files=@file2.pdf" \
  -o merged.pdf
```

### Image → PDF

```bash
curl -u admin:password \
  -X POST http://localhost:7100/forms/chromium/convert/html \
  -F "files=@index.html" \
  -o output.pdf
```

> Wrap gambar dalam HTML sederhana, lalu convert via Chromium.

---

## Konfigurasi

Basic auth API aktif dari `docker-compose.yml` dengan:
- `API_ENABLE_BASIC_AUTH=true`
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
4. Pastikan mapping port `7100:3000` aktif di `docker-compose.yml`
5. Tambahkan reverse proxy/WAF hanya jika butuh rate limit atau proteksi edge

---

## Dokumentasi Lengkap

- [Gotenberg Docs](https://gotenberg.dev/docs)
