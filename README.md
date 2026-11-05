# KatalogKu API — Microservice dengan PocketBase

Backend aplikasi KatalogKu (praktikum PAWM / II3140) yang disusun sebagai **microservice**
tanpa menulis kode server. Semua tool gratis & open source:

| Komponen | Tool | Lisensi |
|---|---|---|
| product-service, account-service | [PocketBase](https://pocketbase.io) v0.40.5 | MIT |
| API gateway | Nginx 1.28 | BSD-2 |
| Orkestrasi | Docker Compose | Apache-2.0 |
| Uji API | REST Client (VS Code) / Postman / Bruno | MIT / gratis |
| Akses publik (opsional) | Cloudflare Tunnel (`cloudflared`) | Apache-2.0 |

## Arsitektur

```
 Aplikasi / Postman
        │  http://IP_SERVER:8080/api/...
        ▼
 ┌─────────────┐   /api/products    ┌──────────────────┐
 │  gateway    │ ─────────────────► │ product-service  │ → SQLite (volume product_data)
 │  (Nginx)    │   /api/auth/...    ├──────────────────┤
 │  :8080      │ ─────────────────► │ account-service  │ → SQLite (volume account_data)
 └─────────────┘   /api/favorites   └──────────────────┘
```

Setiap service punya database sendiri (database-per-service). Favorit hanya menyimpan
`product_id`; detail produk diambil aplikasi dari product-service.

## Struktur folder

```
katalogku-pocketbase/
├── docker-compose.yml        # 3 service + tunnel (opsional)
├── .env.example              # TUNNEL_TOKEN (opsional)
├── pocketbase/Dockerfile     # image PocketBase (amd64 & arm64/STB)
├── gateway/nginx.conf        # pemetaan URL /api/... → service
├── pb_schema/                # skema koleksi siap import
│   ├── product_collections.json
│   └── account_collections.json
├── seed.http                 # isi 5 produk contoh
├── api.http                  # uji semua endpoint (REST Client)
└── postman/                  # koleksi + environment Postman
```

## Menjalankan

```bash
# 1. Jalankan semua service (pertama kali butuh internet untuk build image)
docker compose up -d --build
docker compose ps

# 2. Buat superuser (admin dashboard) di kedua service
docker compose exec product-service \
  pocketbase superuser upsert admin@gmail.com admin123 --dir=/pb_data
docker compose exec account-service \
  pocketbase superuser upsert admin@gmail.com admin123 --dir=/pb_data
```

3. **Periksa skema otomatis** — pada database baru koleksi diimpor oleh migrasi. Impor manual hanya jika diperlukan dan setelah memeriksa data yang sudah ada:
   - `http://IP_SERVER:8091/_/` → `pb_schema/product_collections.json`
   - `http://IP_SERVER:8092/_/` → `pb_schema/account_collections.json`
4. **Periksa data contoh** — lima produk diisi otomatis pada database baru. Jika perlu mengisi manual, buka `seed.http` di VS Code (ekstensi REST Client), ganti
   `@pbUrl` dan password, lalu klik *Send Request* satu per satu.
5. **Uji API lewat gateway** — buka `api.http`, ganti `@baseUrl`, jalankan berurutan
   (register → login → favorit). Alternatif: import folder `postman/` ke Postman dan pilih
   environment *Lokal*, *ServerLAN*, atau *Cloudflare*.

Cek cepat: `curl http://localhost:8080/api/health` dan `curl http://localhost:8080/api/products`.

## Endpoint (lewat gateway :8080)

| Method | Endpoint | Auth | Keterangan |
|---|---|---|---|
| GET | `/api/products?perPage=30` | - | daftar produk `{ items, page, totalItems }` |
| GET | `/api/products?filter=title~'kopi'` | - | cari produk |
| GET | `/api/products/:id` | - | detail produk |
| POST | `/api/auth/records` | - | register `{ name, email, password, passwordConfirm }` |
| POST | `/api/auth/auth-with-password` | - | login `{ identity, password }` → `{ token, record }` |
| GET | `/api/favorites` | Bearer | favorit milik user login |
| POST | `/api/favorites` | Bearer | `{ user, product_id }` |
| DELETE | `/api/favorites/:recordId` | Bearer | hapus favorit |

Tambah/ubah produk dilakukan admin lewat dashboard atau `seed.http` (API Rule create = superuser saja).

## Akses dari internet (opsional)

- **Quick tunnel** (tanpa akun, URL berubah tiap dijalankan):
  `docker run --rm --network host cloudflare/cloudflared:latest tunnel --no-autoupdate --url http://localhost:8080`
- **Tunnel permanen**: salin token dari dashboard Cloudflare ke `.env` (`TUNNEL_TOKEN=...`),
  lalu `docker compose --profile tunnel up -d`.

## Operasional

```bash
git pull && docker compose up -d --build            # update
docker compose logs -f account-service              # log
docker compose cp product-service:/pb_data ./backup-product   # backup
docker compose cp account-service:/pb_data ./backup-account
```

> **Keamanan:** ganti semua password contoh, jangan commit `.env`, dan jangan membuka port
> 8091/8092 ke internet — cukup gateway (8080) atau tunnel.

## Penyiapan otomatis untuk database baru

Saat `docker compose up -d --build` dijalankan pada database baru, migrasi menyiapkan koleksi products, users, favorites, aturan akses, dan lima produk contoh. Jika koleksi terkait sudah ada, migrasi melewati penyiapan awal agar tidak menggandakan produk atau menimpa koleksi tersebut. Migrasi hanya dijalankan sekali untuk setiap database.

Akun admin tidak disimpan di GitHub. Setelah layanan berjalan, buat superuser pada product-service dan account-service dengan langkah pada bagian Menjalankan di atas. Gunakan password yang disepakati lewat jalur pribadi, bukan password contoh. Data yang ditambahkan di laptop tidak otomatis ikut Git; database baru di STB memperoleh lima produk contoh dari migrasi.

Perubahan koleksi berikutnya harus diekspor dan disiapkan sebagai migrasi baru secara sengaja; pembuatan migrasi otomatis dari dashboard dinonaktifkan untuk menjaga file migrasi yang dipasang hanya-baca.
