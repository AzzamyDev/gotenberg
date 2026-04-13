# PDF Service — Gotenberg + Traefik

Service untuk generate PDF dari HTML/URL dan merge PDF/gambar, menggunakan **Gotenberg** sebagai engine dan **Traefik** sebagai reverse proxy dengan rate limiting. Autentikasi hanya di **Gotenberg** (basic auth API).

---

## Stack

| Komponen | Fungsi |
|---|---|
| Gotenberg 8 | PDF engine (Chromium + LibreOffice) |
| Traefik v3.3 | Reverse proxy, rate limit |

---

## Fitur

- HTML → PDF (simple & complex dengan JS)
- URL → PDF
- Merge PDF
- Image → PDF
- Rate limiting per IP
- Basic Auth di Gotenberg (`GOTENBERG_USER` / `GOTENBERG_PASS`)
- Port & credentials via `.env`

---

## Struktur Folder

```
project/
├── docker-compose.yml
├── .env                  ← buat dari .env.example
├── traefik/
│   └── dynamic.yml       ← routing & middleware config
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
EXPOSE_PORT=7100
DOCKER_SOCK=/var/run/docker.sock
GOTENBERG_USER=admin
GOTENBERG_PASS=your_strong_password
```

### 2. Jalankan

```bash
docker compose up -d
```

### 3. Cek Status

```bash
curl -u admin:your_password http://localhost:7100/health
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
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7100/health
```

Dengan kredensial dari `.env` → harus **`200`**:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -u admin:your_password \
  http://localhost:7100/health
```

### POST URL → PDF

Tanpa auth → **`401`**:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -X POST http://localhost:7100/forms/chromium/convert/url \
  -F "url=https://example.com"
```

Dengan auth → **`200`** jika konversi berhasil (bisa **`4xx`** kalau situs / Chromium menolak):

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -u admin:your_password \
  -X POST http://localhost:7100/forms/chromium/convert/url \
  -F "url=https://example.com"
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
  -F "url=https://example.com" \
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

### Rate Limiting (`traefik/dynamic.yml`)

| Setting | Default | Keterangan |
|---|---|---|
| `average` | 10 | Max request/detik per IP |
| `burst` | 20 | Max spike request |
| `period` | 1s | Window perhitungan |

Aktifkan basic auth API dengan `API_ENABLE_BASIC_AUTH=true` di `docker-compose.yml` (bukan `GOTENBERG_API_ENABLE_BASIC_AUTH`). User/password tetap `GOTENBERG_API_BASIC_AUTH_USERNAME` / `GOTENBERG_API_BASIC_AUTH_PASSWORD` dari `.env`.

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
4. Untuk scale: tambah replica Gotenberg dan update `dynamic.yml` dengan multiple servers
5. Bila perlu proteksi edge, tambahkan auth di Traefik, WAF, atau jaringan privat

---

## Dokumentasi Lengkap

- [Gotenberg Docs](https://gotenberg.dev/docs)
- [Traefik Docs](https://doc.traefik.io/traefik)
