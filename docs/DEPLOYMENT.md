# Panduan Deployment HAULOPS v2.0

## Arsitektur Deployment

```
┌─────────────────────────────────────────────────────────────┐
│                      GITHUB REPOSITORY                       │
│                                                              │
│  Branch: dev  ──── Pull Request ────► Branch: main           │
│     │                                      │                 │
│  CI Only                           CI + Auto Deploy          │
└─────────────────────────────────────────────────────────────┘
                                            │
              ┌─────────────────────────────┤
              │                             │
              ▼                             ▼
  ┌───────────────────┐         ┌───────────────────┐
  │   🚂 RAILWAY       │         │   🎨 RENDER        │
  │   (Frontend)       │         │   (Backend)        │
  │                   │         │                   │
  │  React + Vite     │◄───────►│  Express + Prisma  │
  │  (Nginx Docker)   │  API    │  (Docker)          │
  │                   │  calls  │                   │
  └───────────────────┘         └────────┬──────────┘
                                         │
                                         │ DATABASE_URL
                                         ▼
                              ┌───────────────────┐
                              │   🟩 SUPABASE      │
                              │   (Database)       │
                              │                   │
                              │  PostgreSQL 15    │
                              │  Free Tier        │
                              └───────────────────┘
```

| Layer | Platform | Free Tier | Catatan |
|-------|----------|-----------|---------|
| **Frontend** | Railway | ✅ $5 kredit awal | React/Vite disajikan via Nginx |
| **Backend** | Render | ✅ Permanen (dengan sleep) | Express + Prisma, sleep 15 mnt jika idle |
| **Database** | Supabase | ✅ Permanen | PostgreSQL, 500MB storage, 2 project |

---

## File Konfigurasi yang Dibuat

| File | Lokasi | Fungsi |
|------|--------|--------|
| `Dockerfile.prod` | `scaf/packages/server/` | Build production backend (multi-stage) |
| `Dockerfile.prod` | `scaf/packages/web/` | Build production frontend → Nginx |
| `nginx.conf` | `scaf/packages/web/` | Nginx config untuk SPA React Router |
| `render.yaml` | `/` (root repo) | Blueprint Render — auto-detect service |
| `railway.json` | `/` (root repo) | Config Railway — Dockerfile builder |
| `.env.example` | `scaf/packages/server/` | Panduan environment variables |
| `schema.prisma` | `scaf/packages/server/prisma/` | Tambah `directUrl` untuk Supabase |
| `server-tests.yml` | `.github/workflows/` | CI/CD GitHub Actions |

---

## Langkah-Langkah Deployment

### Step 1 — Supabase (Database) 🟩

> Database harus disiapkan pertama karena backend membutuhkan `DATABASE_URL`.

1. Buat akun di [https://supabase.com](https://supabase.com)
2. Klik **New Project**, isi:
   - **Name**: `haulops`
   - **Database Password**: buat password yang kuat, **simpan baik-baik**
   - **Region**: `Southeast Asia (Singapore)` ← pilih yang terdekat
3. Tunggu project selesai dibuat (~2 menit)
4. Klik tombol hijau **"Connect"** di bar atas dashboard (menu lama "Settings →
   Database" sudah tidak ada). Pilih tab **ORMs → Prisma** untuk melihat kedua URL
   langsung, atau tab **Connection string** (sub-tab Transaction/Session pooler).
5. Catat **dua URL** berikut (ganti `[password]` dengan Database Password):

   **`DATABASE_URL`** — Transaction Pooler (untuk runtime/production):
   ```
   postgresql://postgres.[ref]:[password]@aws-0-ap-southeast-1.pooler.supabase.com:6543/postgres?pgbouncer=true&connection_limit=1
   ```

   **`DIRECT_URL`** — Session Pooler atau Direct (untuk Prisma migrate):
   ```
   postgresql://postgres.[ref]:[password]@aws-0-ap-southeast-1.pooler.supabase.com:5432/postgres
   ```

> [!NOTE]
> Tabel database **tidak perlu dibuat manual**. Prisma akan otomatis menjalankan `migrate deploy` saat backend pertama kali start di Render.

---

### Step 2 — Render (Backend) 🎨

> Backend dideploy ke Render menggunakan `render.yaml` Blueprint.

1. Buat akun di [https://render.com](https://render.com) (login via GitHub)
2. Klik **New → Blueprint Instance**
3. Hubungkan GitHub repository HAULOPS
4. Render otomatis mendeteksi file `render.yaml` di root repo
5. Klik **Apply** — Render akan mulai mem-build Docker image
6. Setelah service terbuat, buka **Environment** di dashboard Render, isi:

   | Key | Value |
   |-----|-------|
   | `DATABASE_URL` | Transaction Pooler URL dari Supabase (port 6543) |
   | `DIRECT_URL` | Session Pooler URL dari Supabase (port 5432) — wajib untuk migrate |
   | `JWT_SECRET` | *(auto-generated oleh Render — dipakai untuk menandatangani token JWT)* |
   | `JWT_EXPIRES_IN` | `8h` |
   | `NODE_ENV` | `production` |
   | `PORT` | `4001` |
   | `CORS_ORIGIN` | *(isi setelah Railway deploy — URL frontend Railway)* |
   | `ADMIN_USERNAME` | `admin` *(akun admin awal)* |
   | `ADMIN_PASSWORD` | *(password kuat — wajib diisi; bila kosong dipakai `admin`)* |

   > [!NOTE]
   > **Akun admin awal dibuat otomatis.** Saat DB masih kosong, backend membuat
   > 1 akun admin + 1 branch (`Kantor Pusat`) saat start — cukup untuk login
   > pertama. Bersifat *idempotent*: begitu sudah ada user, tidak dibuat ulang.
   > Tidak perlu menjalankan seed manual. Ganti password admin lewat **Settings**
   > setelah login pertama.

7. Klik **Save Changes** → Render otomatis trigger redeploy
8. Tunggu status menjadi **Live**, catat URL backend:
   ```
   https://haulops-server.onrender.com
   ```

> [!WARNING]
> Di Free Tier Render, server akan **tidur (sleep)** setelah 15 menit tidak ada request. Request pertama setelah sleep akan butuh waktu ~30 detik untuk bangun. Ini normal untuk free tier.

---

### Step 3 — Railway (Frontend) 🚂

> Frontend dideploy ke Railway menggunakan `railway.json`.

1. Buat akun di [https://railway.app](https://railway.app) (login via GitHub)
2. Klik **New Project → Deploy from GitHub Repo**
3. Pilih repository HAULOPS
4. Railway otomatis mendeteksi `railway.json` di root repo
5. Buka tab **Variables**, tambahkan:

   | Key | Value |
   |-----|-------|
   | `VITE_API_URL` | `https://haulops-server.onrender.com` |

   > [!IMPORTANT]
   > Isi **origin backend saja** (tanpa `/api/v1` di belakang). Kode frontend
   > sudah menambahkan `/api/v1/...` sendiri. Nilai ini di-inject saat **build**
   > (build arg di `Dockerfile.prod`), jadi setiap ganti nilai → perlu redeploy.
   > Dikosongkan/absen di dev → frontend memakai path relatif lewat proxy Vite.

6. Klik **Deploy** → Railway akan build Docker image dan deploy
7. Setelah selesai, catat URL frontend:
   ```
   https://haulops-web.up.railway.app
   ```

---

### Step 4 — Update CORS di Render 🔗

Setelah Railway memberikan URL frontend, update environment variable `CORS_ORIGIN` di Render:

1. Buka Render Dashboard → Service `haulops-server`
2. Buka tab **Environment**
3. Set `CORS_ORIGIN` = `https://haulops-web.up.railway.app`
4. Klik **Save** → Render otomatis redeploy backend

---

### Step 5 — Verifikasi Akhir ✅

Cek seluruh sistem berjalan dengan baik:

- [ ] `GET https://haulops-server.onrender.com/api/health` → mengembalikan `200 OK`
- [ ] Buka URL Railway di browser → halaman login muncul
- [ ] Login dengan **akun admin awal** (`ADMIN_USERNAME` / `ADMIN_PASSWORD`, default `admin`/`admin`) → berhasil masuk dashboard
- [ ] **Segera ganti password admin** lewat menu Settings
- [ ] Data tampil dari Supabase (cek di Supabase Table Editor)

---

## Alur Update Sistem (CI/CD)

### Skenario: Developer Push Kode Baru

```
git add .
git commit -m "feat: tambah fitur X"
git push origin main
```

```
GitHub menerima push
    │
    ▼
GitHub Actions berjalan otomatis:
  ├── Job 1: Backend TypeCheck      ── ~1 menit
  └── Job 2: Frontend Build (Vite)  ── ~2 menit
    │
    ▼ (jika semua pass)
  ├── 🚂 Railway  → Detect push → Re-build → Redeploy Frontend
  └── 🎨 Render   → Detect push → Re-build → Redeploy Backend
                                      └── prisma migrate deploy (otomatis)
```

### Strategi 3 Branch (Environment)

Alur promosi: **`dev` → `staging` → `main`**.

| Branch | Peran | GitHub Actions | Deploy (Railway + Render) |
|--------|-------|---------------|---------------------------|
| `dev` | Integrasi fitur | ✅ CI (typecheck + build) | ❌ Tidak deploy |
| `staging` | Pra-produksi / QA | ✅ CI | ✅ Deploy ke **environment staging** |
| `main` | Produksi | ✅ CI | ✅ Deploy ke **environment produksi** |
| PR ke `staging` / `main` | Review | ✅ CI | ❌ Tidak deploy |

> [!IMPORTANT]
> Agar `staging` benar-benar men-deploy terpisah dari produksi, buat **service
> terpisah** di tiap platform yang dipatok ke branch `staging`:
> - **Render**: service backend kedua (Settings → Branch = `staging`), dengan
>   env sendiri (idealnya `DATABASE_URL`/`DIRECT_URL` ke **project/DB Supabase
>   staging** yang terpisah agar data produksi aman).
> - **Railway**: service frontend kedua (Deploy branch = `staging`), `VITE_API_URL`
>   diarahkan ke origin backend staging.
> - **Supabase**: project/database staging terpisah (opsional tapi disarankan).
>
> Tanpa service terpisah, hanya `main` yang otomatis ter-deploy; `staging`/`dev`
> tetap menjalankan CI sebagai gerbang kualitas.

---

## ⚠️ Perubahan Schema Database (Wajib Dibaca)

Jika ada perubahan pada file `prisma/schema.prisma` (tambah tabel, ubah kolom, dll), **wajib** jalankan perintah ini di lokal sebelum push ke `main`:

```bash
cd scaf/packages/server

# Buat file migrasi baru
npx prisma migrate dev --name deskripsi_perubahan

# Contoh:
npx prisma migrate dev --name tambah_kolom_keterangan_di_rit
```

Perintah ini akan membuat file SQL di `prisma/migrations/`. File ini yang dibaca oleh Prisma di Render saat `prisma migrate deploy` otomatis dijalankan.

> [!CAUTION]
> Jangan pernah push perubahan `schema.prisma` ke `main` tanpa file migrasi yang sesuai. Backend akan crash saat start karena skema database tidak sinkron dengan kode.

---

## Environment Variables Lengkap

### Backend (Render)

| Variable | Contoh Nilai | Wajib | Keterangan |
|----------|-------------|-------|------------|
| `DATABASE_URL` | `postgresql://...@pooler.supabase.com:6543/postgres?pgbouncer=true` | ✅ | Transaction Pooler Supabase |
| `DIRECT_URL` | `postgresql://...@pooler.supabase.com:5432/postgres` | ✅ | Untuk `prisma migrate deploy` |
| `JWT_SECRET` | *(random string 32+ karakter)* | ✅ | Auto-generate oleh Render |
| `JWT_EXPIRES_IN` | `8h` | ✅ | Durasi token valid |
| `NODE_ENV` | `production` | ✅ | Mode production |
| `PORT` | `4001` | ✅ | Port server |
| `CORS_ORIGIN` | `https://haulops-web.up.railway.app` | ✅ | URL frontend Railway |

### Frontend (Railway)

| Variable | Contoh Nilai | Wajib | Keterangan |
|----------|-------------|-------|------------|
| `VITE_API_URL` | `https://haulops-server.onrender.com` | ✅ | **Origin** backend Render (tanpa `/api/v1`); di-inject saat build |

---

## Monitoring & Troubleshooting

### Cek Log

| Platform | Cara Cek Log |
|----------|-------------|
| **Render** | Dashboard → Service → Logs tab |
| **Railway** | Dashboard → Service → Deployments → View Logs |
| **Supabase** | Dashboard → Database → Query Editor atau Table Editor |

### Masalah Umum

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|---------------------|--------|
| Backend tidak merespons | Render sedang sleep (free tier) | Tunggu 30 detik, coba lagi |
| Login gagal (401) | `JWT_SECRET` tidak ter-set | Cek env var di Render |
| Data tidak muncul | `DATABASE_URL` salah | Cek connection string Supabase |
| Frontend gagal build | `VITE_API_URL` belum diset | Tambah env var di Railway |
| Halaman 404 saat refresh | Nginx config belum apply | Pastikan `nginx.conf` ikut di-copy |
| Prisma migrate gagal | `DIRECT_URL` menggunakan pooler port 6543 | Ganti ke port 5432 (Session Pooler) |

---

## URL Layanan (isi setelah deploy)

| Layanan | URL |
|---------|-----|
| **Frontend** (Railway) | `https://____________.up.railway.app` |
| **Backend** (Render) | `https://____________.onrender.com` |
| **API Health Check** | `https://____________.onrender.com/api/v1/health` |
| **Supabase Dashboard** | `https://supabase.com/dashboard/project/____________` |
