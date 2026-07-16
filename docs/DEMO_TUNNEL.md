# Panduan Menyalakan Demo HAULOPS (Cloudflare Tunnel)

Cara menjalankan aplikasi HAULOPS **lokal** dan membukanya ke internet lewat
**Cloudflare Tunnel** — untuk demo sementara (mis. ke management), **tanpa hosting,
tanpa kartu kredit**.

> Aplikasi berjalan di komputer Anda; tunnel memberi 1 URL publik HTTPS. **Komputer
> harus menyala & semua proses jalan** selama demo diakses.

---

## Prasyarat (sekali saja)
- **Docker Desktop** terpasang (untuk database PostgreSQL).
- **Node.js 20** terpasang.
- **cloudflared** ada di: `C:\Users\LENOVO\cloudflared.exe`
  *(jika hilang, unduh ulang: https://github.com/cloudflare/cloudflared/releases/latest → `cloudflared-windows-amd64.exe`, rename jadi `cloudflared.exe`)*.

---

## Menyalakan sesi demo (4 komponen)

Buka **4 jendela terminal terpisah**, biarkan semuanya **tetap terbuka** selama demo.

### 1) Database (Docker)
Pastikan Docker Desktop sudah jalan, lalu:
```powershell
docker start haulops-db
```
Cek sehat: `docker ps` → status `haulops-db` harus `Up ... (healthy)`.

### 2) Backend (port 4001)
```powershell
cd d:\AI\haulops\scaf\packages\server
npm run dev
```
Tunggu sampai muncul `Server listening on http://localhost:4001`.

### 3) Frontend (port 5173)
```powershell
cd d:\AI\haulops\scaf\packages\web
npm run dev
```
Tunggu sampai muncul `Local: http://localhost:5173/`.

### 4) Tunnel (URL publik)
```powershell
C:\Users\LENOVO\cloudflared.exe tunnel --url http://localhost:5173 --http-host-header localhost:5173
```
Di output akan muncul baris seperti:
```
https://xxxx-xxxx-xxxx.trycloudflare.com
```
**Itulah URL yang dibagikan ke management.**

> ⚠️ Flag `--http-host-header localhost:5173` **wajib** — tanpa itu Vite menolak
> request dari domain tunnel ("Blocked request. This host is not allowed").

---

## Login
- URL: (dari langkah 4)
- Username: `admin`
- Password: `password`

Data contoh sudah terisi: **Project NPM, Juni 2026** (lihat menu **Daily Report**,
**Analytic**, **Dashboard**, **Master Data → Budget & Target**).

---

## Menghentikan sesi
- Tutup masing-masing jendela terminal (atau tekan `Ctrl + C` di tiap jendela).
- Database boleh dibiarkan / hentikan dengan `docker stop haulops-db`.

---

## Catatan penting
- **URL berubah setiap tunnel dijalankan ulang** (quick tunnel = acak). Ambil URL
  terbaru dari output langkah 4 tiap kali start.
- **URL bersifat publik** — siapa pun yang punya link bisa membuka. Bagikan hanya
  ke pihak yang dituju. Ini demo data dummy, jadi risiko rendah.
- **Komputer harus menyala** selama diakses. Jika PC sleep/restart, jalankan ulang
  langkah 1–4.
- Untuk **URL stabil/custom** (tidak berubah), perlu **named tunnel** + domain
  sendiri di Cloudflare (gratis) — lihat bagian di bawah.

---

## Troubleshooting

| Gejala | Solusi |
|---|---|
| Tunnel tampil "Blocked request ... host not allowed" | Pastikan pakai `--http-host-header localhost:5173` di langkah 4 |
| `Port 5173 is in use` / `4001 in use` | Proses lama masih jalan: `npx kill-port 5173` atau `npx kill-port 4001`, lalu start ulang |
| Backend error connect DB | Docker Desktop belum nyala / container mati → `docker start haulops-db` |
| Halaman putih / API gagal | Pastikan backend (4001) & frontend (5173) dua-duanya jalan sebelum start tunnel |
| `cloudflared` tak dikenal | Cek path `C:\Users\LENOVO\cloudflared.exe` benar |

---

## (Opsional) URL stabil & custom — Named Tunnel

Quick tunnel gampang tapi URL-nya acak & berubah. Untuk URL **tetap** selama masa
demo (mis. 2 minggu), pakai **named tunnel** (butuh akun Cloudflare gratis; kalau
mau URL custom seperti `haulops.domainanda.com`, perlu punya domain di Cloudflare):

```powershell
# 1. Login (buka browser, pilih domain di Cloudflare) — sekali saja
C:\Users\LENOVO\cloudflared.exe tunnel login

# 2. Buat named tunnel
C:\Users\LENOVO\cloudflared.exe tunnel create haulops-demo

# 3. Arahkan subdomain ke tunnel (perlu domain di Cloudflare)
C:\Users\LENOVO\cloudflared.exe tunnel route dns haulops-demo haulops.domainanda.com

# 4. Jalankan (URL tetap: https://haulops.domainanda.com)
C:\Users\LENOVO\cloudflared.exe tunnel run --url http://localhost:5173 --http-host-header localhost:5173 haulops-demo
```

Tanpa domain sendiri, tetap gunakan **quick tunnel** (URL acak) seperti di atas.
