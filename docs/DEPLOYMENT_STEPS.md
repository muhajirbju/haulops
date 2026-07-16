# Panduan Setup Deployment HAULOPS — Langkah Detail per Platform

Panduan klik-demi-klik untuk deploy **produksi** (branch `main`).
Arsitektur: **Supabase** (database) → **Render** (backend) → **Railway** (frontend).

Urutan wajib: **Supabase dulu** (backend butuh URL-nya), lalu Render, lalu Railway,
terakhir update CORS. Siapkan ~30–45 menit.

> Ringkasan alur & troubleshooting umum ada di `docs/DEPLOYMENT.md`. Dokumen ini
> fokus ke langkah rinci tiap platform.

---

## Persiapan (5 menit)

1. Pastikan punya akun **GitHub** dengan akses ke repo `muhajirbju/haulops`.
2. Siapkan **satu password kuat** untuk database (akan dipakai di Supabase) — catat di tempat aman.
3. Siapkan **satu password kuat** untuk admin aplikasi (akan jadi `ADMIN_PASSWORD` di Render).
4. Semua kode sudah ada di branch `main` — tidak perlu push apa pun lagi.

Nilai yang akan Anda kumpulkan sambil jalan (isi saat dapat):

| Nama | Dari | Contoh |
|---|---|---|
| `DATABASE_URL` | Supabase (pooler 6543) | `postgresql://postgres.abcd:...@aws-0-ap-southeast-1.pooler.supabase.com:6543/postgres?pgbouncer=true` |
| `DIRECT_URL` | Supabase (session pooler 5432) | `postgresql://postgres.abcd:...@aws-0-ap-southeast-1.pooler.supabase.com:5432/postgres` |
| URL Backend | Render | `https://haulops-server.onrender.com` |
| URL Frontend | Railway | `https://haulops-web.up.railway.app` |

---

## STEP 1 — Supabase (Database) 🟩

### 1.1 Buat project
1. Buka **https://supabase.com** → **Start your project** → login **with GitHub**.
2. **New project**.
3. Isi:
   - **Organization**: pilih/ buat (mis. `BJU`).
   - **Name**: `haulops-prod`.
   - **Database Password**: klik **Generate a password** (atau isi password kuat Anda) → **SALIN & SIMPAN**. Ini password DB, dipakai di connection string.
   - **Region**: **Southeast Asia (Singapore)** — `ap-southeast-1`.
   - **Plan**: Free.
4. **Create new project** → tunggu ~2 menit sampai status hijau (provisioning).

### 1.2 Ambil connection string (bagian paling penting)
1. Di halaman project, klik tombol **Connect** (atas), atau **Project Settings (ikon gerigi) → Database**.
2. Cari bagian **Connection string** / **Connection pooling**. Ada beberapa mode — kita pakai **dua**:

   **a) `DATABASE_URL` — mode "Transaction pooler" (port 6543):**
   ```
   postgresql://postgres.[REF]:[PASSWORD]@aws-0-ap-southeast-1.pooler.supabase.com:6543/postgres?pgbouncer=true
   ```
   Tambahkan di belakang: `&connection_limit=1` → jadi:
   ```
   ...:6543/postgres?pgbouncer=true&connection_limit=1
   ```

   **b) `DIRECT_URL` — mode "Session pooler" (port 5432):**
   ```
   postgresql://postgres.[REF]:[PASSWORD]@aws-0-ap-southeast-1.pooler.supabase.com:5432/postgres
   ```

3. **Ganti `[PASSWORD]`** pada kedua URL dengan Database Password dari langkah 1.3. `[REF]` sudah terisi otomatis (kode project Anda).

> [!CAUTION]
> **JANGAN pakai "Direct connection"** (`db.[ref].supabase.co:5432`) untuk `DIRECT_URL`.
> Host itu **IPv6-only**, dan Render **tidak mendukung IPv6 keluar** → `prisma migrate
> deploy` akan gagal. Gunakan **Session pooler** (host `...pooler.supabase.com:5432`)
> seperti di atas. Keduanya (`DATABASE_URL` 6543 & `DIRECT_URL` 5432) memakai host
> pooler yang sama, beda port saja.

> [!NOTE]
> Tabel dibuat otomatis oleh `prisma migrate deploy` saat backend pertama start.
> **Tidak perlu** bikin tabel manual. Akun admin juga dibuat otomatis (lihat Step 2).

---

## STEP 2 — Render (Backend) 🎨

### 2.1 Buat service dari Blueprint
1. Buka **https://render.com** → **Get Started** / login **with GitHub**.
2. Klik **New +** (kanan atas) → **Blueprint**.
3. **Connect a repository** → pilih `muhajirbju/haulops`.
   - Jika repo tak muncul: **Configure account** → beri Render akses ke repo/organisasi `muhajirbju`.
4. Render otomatis membaca **`render.yaml`** di root → menampilkan service **`haulops-server`**.
5. Klik **Apply** / **Create Services**.

### 2.2 Isi Environment Variables
Render akan meminta nilai untuk env yang `sync:false`. Buka service **haulops-server → Environment** dan isi:

| Key | Value | Catatan |
|---|---|---|
| `DATABASE_URL` | *(dari Supabase, pooler 6543)* | wajib |
| `DIRECT_URL` | *(dari Supabase, session pooler 5432)* | wajib untuk migrate |
| `ADMIN_PASSWORD` | *(password admin kuat Anda)* | login pertama pakai ini |
| `CORS_ORIGIN` | `https://placeholder` sementara | **diisi ulang di Step 4** setelah Railway jadi |

Yang **sudah otomatis** (tak perlu diisi): `JWT_SECRET` (auto-generate), `JWT_EXPIRES_IN=8h`,
`NODE_ENV=production`, `PORT=4001`, `ADMIN_USERNAME=admin`.

6. **Save Changes** → Render otomatis build ulang.

### 2.3 Tunggu Live & catat URL
1. Buka tab **Logs**. Build memakai Docker (~3–6 menit). Yang sehat:
   - `prisma migrate deploy` → "All migrations have been successfully applied."
   - `[bootstrap] Akun admin 'admin' dibuat` (hanya muncul saat DB pertama kali kosong).
   - `Server listening on http://localhost:4001`.
2. Status berubah jadi **Live**. Catat URL di atas halaman, mis:
   ```
   https://haulops-server.onrender.com
   ```
3. Tes cepat di browser: buka `https://<url-render>/api/health` → harus `{"status":"ok"...}` / `200`.

> [!WARNING]
> Free tier Render **tidur setelah 15 menit idle**. Request pertama setelah tidur
> butuh ~30–50 detik untuk bangun. Normal.

---

## STEP 3 — Railway (Frontend) 🚂

### 3.1 Buat project
1. Buka **https://railway.app** → login **with GitHub**.
2. **New Project** → **Deploy from GitHub repo** → pilih `muhajirbju/haulops`.
3. Railway membuat service dari repo. Buka service tsb → **Settings**.

### 3.2 Set Root Directory & Builder (WAJIB untuk monorepo)
Di **Settings → Build** (atau Source):
1. **Root Directory**: `scaf/packages/web`
   *(penting — di sinilah `Dockerfile.prod`, `package.json`, `railway.json` frontend berada. Tanpa ini build gagal di `COPY`.)*
2. **Builder**: **Dockerfile** (railway.json di folder itu sudah menetapkan `dockerfilePath: Dockerfile.prod`; kalau ditanya, isi `Dockerfile.prod`).

### 3.3 Set Variable (build arg)
Buka tab **Variables** → **New Variable**:

| Key | Value |
|---|---|
| `VITE_API_URL` | `https://haulops-server.onrender.com` |

> [!IMPORTANT]
> Isi **origin backend Render saja — TANPA `/api/v1`** di belakang. Kode frontend
> menambahkan `/api/v1/...` sendiri. Nilai ini dipakai saat **build** (build arg),
> jadi setiap ganti nilai → harus **Redeploy**.

### 3.4 Generate domain publik & set port
1. **Settings → Networking** → **Generate Domain**.
2. Bila diminta port, isi **80** (Nginx di container dengar di port 80).
3. Railway build & deploy (~2–4 menit). Catat URL, mis:
   ```
   https://haulops-web.up.railway.app
   ```

---

## STEP 4 — Sambungkan CORS 🔗

Backend harus mengizinkan origin frontend.
1. Buka **Render → haulops-server → Environment**.
2. Ubah `CORS_ORIGIN` = URL Railway (persis, tanpa slash di akhir):
   ```
   https://haulops-web.up.railway.app
   ```
3. **Save Changes** → Render redeploy otomatis (~1–2 menit).

---

## STEP 5 — Verifikasi Akhir ✅

Cek berurutan:
- [ ] `https://<render>/api/health` → `200`.
- [ ] Buka URL Railway di browser → halaman **login** muncul (bukan layar putih/error).
- [ ] Login: username `admin`, password = `ADMIN_PASSWORD` yang Anda set.
- [ ] Masuk dashboard, data tampil (menu Master, dll).
- [ ] **Segera ganti password admin**: menu **Settings → Ubah Password**.
- [ ] (Opsional) Cek di **Supabase → Table Editor** bahwa tabel & user admin ada.

Jika semua ✅ → **produksi live**. 🎉

---

## Update Berikutnya (CI/CD Otomatis)

Setelah setup awal, cukup **push ke `main`** (via promosi `dev → staging → main`):
- GitHub Actions jalan (typecheck + build) sebagai gerbang.
- Render & Railway menerima webhook → build & redeploy otomatis.
- `prisma migrate deploy` jalan otomatis saat backend restart (pastikan file migrasi
  sudah ada di `prisma/migrations/` sebelum push — lihat `docs/DEPLOYMENT.md`).

---

## (Opsional) Environment Staging Terpisah

Agar branch `staging` men-deploy terpisah dari produksi, ulangi Step 1–4 dengan
resource **kedua**, dipatok ke branch `staging`:
- Supabase: project/DB **staging** terpisah (agar data produksi aman saat uji coba).
- Render: service backend kedua, **Settings → Branch = `staging`**, env sendiri.
- Railway: service frontend kedua, **Settings → Branch = `staging`**, `VITE_API_URL`
  → origin backend staging.

Tanpa ini, hanya `main` yang auto-deploy; `staging`/`dev` tetap menjalankan CI.

---

## Troubleshooting Cepat

| Gejala | Penyebab | Solusi |
|---|---|---|
| Backend crash saat start, log `migrate` error IPv6/timeout | `DIRECT_URL` pakai host direct (`db.*.supabase.co`) | Ganti ke **Session pooler** (`*.pooler.supabase.com:5432`) |
| Frontend build gagal di Railway, `COPY package.json` not found | Root Directory belum diset | Set **Root Directory = `scaf/packages/web`** |
| Halaman Railway putih, network call ke `/api/...` kena 404/HTML | `VITE_API_URL` salah/kosong, atau pakai `/api/v1` di belakang | Set origin backend **tanpa** `/api/v1`, lalu **Redeploy** |
| Login gagal walau password benar | `CORS_ORIGIN` belum di-set ke URL Railway | Update `CORS_ORIGIN` di Render → redeploy |
| `500` di semua API | `DATABASE_URL` salah | Cek connection string & `[PASSWORD]` sudah diganti |
| Refresh halaman dalam (mis. `#/analytic`) → 404 | (tidak berlaku — app pakai hash routing `#/`) | — |
| Backend lambat ~30 dtk sesekali | Render free tier tidur | Normal; request pertama membangunkan server |
