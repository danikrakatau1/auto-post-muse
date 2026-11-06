# Cara Kerja Sistem Autopost Berita Viral (IG + TikTok + Threads)

_Panduan alur membangun sistem autopost berita 5x sehari — dari nol sampai autopilot. Versi publik, tanpa data internal._

## Gambaran Umum

Sistem ini memposting video berita viral secara otomatis **5x sehari** (07:00, 11:00, 15:00, 19:00, 22:00 WIB) ke tiga channel sekaligus — Instagram, TikTok, dan Threads — lewat **Buffer**. Semua konten berupa **video vertikal HD (1080x1920)** dengan caption santai.

Prinsipnya: **sekali jalan, autopilot**. Berita dicari, diverifikasi, dibuatkan videonya, dijadwalkan — semuanya oleh skrip dan cron. Manusia hanya mengawasi dan menetapkan standar.

```
[Berita + footage viral]
        ↓
[Produksi: video vertikal + caption overlay]
        ↓
[Upload video ke hosting → URL publik]
        ↓
[Buffer API] → IG / TikTok / Threads (terjadwal)
        ↓
[Cron] → jalan otomatis tiap slot + monitor breaking news
```

Yang dibutuhkan: akun IG, TikTok, Threads + akun Buffer (gratis) + satu VPS Linux yang online 24/7 + tools gratis (ffmpeg, yt-dlp, Python, cron).

---

## Alur 1 — Menyiapkan Buffer

1. Daftar di `buffer.com` (paket gratis cukup — maks 3 channel).
2. Hubungkan **Instagram**, **TikTok**, dan **Threads** lewat menu Channels.
3. Ambil API key: buka `publish.buffer.com` → **Settings → API** → salin **personal API key** (bukan token developers/apps).
4. Simpan key dengan aman sebagai environment variable — **jangan tulis di dalam script, jangan commit ke repo publik**:
   ```bash
   echo 'export BUFFER_API_KEY="ISI_KEY_DI_SINI"' >> ~/.bashrc
   source ~/.bashrc
   ```

---

## Alur 2 — Menyiapkan Hosting Video

Buffer tidak menerima upload file langsung dari script — dia butuh **URL publik** video. Alurnya:

1. Buat folder video di VPS (`mkdir -p ~/videos`).
2. Jalankan HTTP server (nginx disarankan).
3. Buat URL publik ke folder itu — lewat domain sendiri atau tunnel. Hasil akhirnya:
   ```
   https://<domain-anda>/2026-11-05-0700.mp4
   ```
4. Tes: upload 1 video, buka URL-nya dari HP — video harus bisa diputar.

> ⚠️ **Matikan directory listing!** Server file default biasanya menampilkan daftar semua file di folder. Siapa pun yang tahu URL hosting bisa melihat dan mengunduh SEMUA video Anda. Di nginx: `autoindex off;`.

---

## Alur 3 — Posting lewat Buffer API

Buffer memakai **GraphQL API** di `https://api.buffer.com` (selalu POST, header `Authorization: Bearer <API key>`).

### Melihat channel

```graphql
{ account { organizations { id name } } }
```

```graphql
{ channels(input: { organizationId: "<ORG-ID>" }) { id name service displayName } }
```

### Membuat post

```graphql
mutation {
  createPost(input: {
    text: "Caption di sini"
    channelId: "<ID-CHANNEL>"
    schedulingType: automatic
    mode: customScheduled dueAt: "2026-11-05T00:00:00Z"
    assets: { video: { url: "https://<domain-anda>/2026-11-05-0700.mp4" } }
    metadata: { instagram: { type: reel, shouldShareToFeed: true } }
  }) {
    ... on PostActionSuccess { post { id text status } }
    ... on MutationError { message }
  }
}
```

- `mode: customScheduled` + `dueAt` (UTC `YYYY-MM-DDTHH:MM:SSZ`) = posting terjadwal.
- `mode: shareNow` = posting langsung (untuk breaking news).
- `metadata: { instagram: { type: reel } }` = **HANYA untuk Instagram**.

### Contoh pemakaian (3 panggilan terpisah — jangan digabung!)

```bash
python3 buffer_post.py --channel <ID-IG> --ig-reel \
  --text "Caption IG" --media https://<domain>/2026-11-05-0700.mp4 --at "2026-11-05 07:00"
python3 buffer_post.py --channel <ID-TIKTOK> \
  --text "Caption TikTok" --media https://<domain>/2026-11-05-0700.mp4 --at "2026-11-05 07:00"
python3 buffer_post.py --channel <ID-THREADS> \
  --text "Caption Threads maks 500 karakter" --media https://<domain>/2026-11-05-0700.mp4 --at "2026-11-05 07:00"
```

### Aturan penting

1. **`--ig-reel` HANYA untuk Instagram.** Metadata IG bikin posting TikTok gagal.
2. **Video WAJIB faststart** (moov atom di depan). Tanpa itu Buffer menolak: `"Video could not be read from its URL"`. Mux dulu:
   ```bash
   ffmpeg -i video.mp4 -c copy -movflags +faststart video-fs.mp4
   ```
   Upload file hasil mux, BUKAN file mentah. Maks ±30 MB.
3. **Threads maks 500 karakter** dan **maks 5 "link"** — string mirip domain ("Mojok.co") DIHITUNG sebagai link. Hapus TLD di caption Threads.
4. **JANGAN PERNAH posting test ke channel live.** Uji coba hanya ke draft / akun test terpisah.

---

## Alur 4 — Produksi Konten (per slot)

Urutan kerja setiap slot (~30–60 menit sebelum jam tayang):

1. **Cari berita** — Google News, portal berita besar, pencarian media sosial. Pilih yang **fresh (maks 1–2 hari)** dan sesuai tema: bola/Timnas, kejadian viral CCTV, kuliner viral, fenomena alam; slot malam khusus politik nasional.
2. **Cek duplikat dulu** — buka log harian; JANGAN pakai cerita yang sudah dipakai slot lain hari ini.
3. **Verifikasi (wajib)** — minimal **2 sumber tepercaya**. Kalau belum pasti, tulis **"diduga"**. Jangan menyebar hoaks. Patuhi UU ITE.
4. **Cari footage video asli** — video vertikal HD yang sudah viral soal berita itu (TikTok/IG), download pakai `yt-dlp`. **Bukan dari stasiun TV** — pakai footage warga/uploader, dan **cantumkan atribusi** sumber + akun footage di caption. Kalau tidak ada footage yang cocok: fallback **slideshow foto** via ffmpeg (efek Ken Burns) ukuran 1080x1920.
5. **Caption overlay** — buat gambar teks (PNG transparan) berisi hook berita, misal gaya: background biru dongker, teks putih + highlight kuning, garis kuning di kiri. Maks ~30 karakter per baris. Overlay ke video dengan ffmpeg — cara paling stabil: **pendekatan segmen** (potong video jadi segmen 8 detik, satu overlay sederhana per segmen, lalu gabung dengan `concat`).
6. **Finalisasi & upload** — mux faststart, upload ke hosting (mis. `scp`), catat URL publiknya. Nama file: `YYYY-MM-DD-HHMM.mp4`.
7. **Tulis caption + schedule** — caption santai + hashtag + atribusi. Threads maks 500 karakter. Schedule via script Alur 3 dengan `--at` jam tayang.
8. **Catat ke log harian** (`posting-YYYY-MM-DD.md`): judul, jam tayang, ID post per channel. Log ini dipakai untuk cek duplikat.

---

## Alur 5 — Otomatisasi dengan Cron

Setelah Alur 1–4 bisa jalan manual, bungkus jadi cron supaya autopilot.

| Cron jalan | Slot tayang | Isi kerja |
|---|---|---|
| 06:00 | — | Brief pagi: riset 5–7 topik hari itu |
| 06:30 | 07:00 | Produksi + schedule slot pagi |
| 10:30 | 11:00 | Produksi + schedule slot siang |
| 14:51 | 15:00 | Produksi + schedule slot sore |
| 18:51 | 19:00 | Produksi + schedule slot malam |
| 21:30 | 22:00 | Produksi + schedule slot politik |
| tiap 20 mnt | — | Monitor breaking news |
| tiap 5 mnt | — | Produksi + tayang breaking yang pending |
| 23:00 | — | Rekap harian |

```cron
# Brief pagi + slot harian (WIB)
0 6 * * * /usr/bin/python3 /home/user/autopost/produksi.py --brief
30 6 * * * /usr/bin/python3 /home/user/autopost/produksi.py --slot 07:00
30 10 * * * /usr/bin/python3 /home/user/autopost/produksi.py --slot 11:00
51 14 * * * /usr/bin/python3 /home/user/autopost/produksi.py --slot 15:00
51 18 * * * /usr/bin/python3 /home/user/autopost/produksi.py --slot 19:00
30 21 * * * /usr/bin/python3 /home/user/autopost/produksi.py --slot 22:00 --politik

# Breaking news
*/20 * * * * /usr/bin/python3 /home/user/autopost/monitor.py
*/5 * * * * /usr/bin/python3 /home/user/autopost/producer.py
```

Satu script `produksi.py` bisa menangani semua slot — isinya persis langkah Alur 4. Awalnya cron bisa memanggil script yang **hanya menyiapkan paket + minta persetujuan Anda**; setelah yakin, ubah ke full otomatis.

---

## Alur 6 — Breaking News (lanjutan, opsional)

1. **Monitor (tiap 20 menit):** cari berita via pencarian berita + pencarian sosial.
2. **Filter ketat** — alert HANYA kalau semua lolos:
   - Kejadian dalam 3 jam terakhir (bukan basi/republish)
   - Skala BESAR (bencana, kecelakaan besar, politik penting, viral nasional)
   - Terverifikasi minimal 2 sumber tepercaya
   - BUKAN gosip artis / hewan unik / konten Malaysia / hoaks
3. **Cek duplikat:** sudah pernah di-alert 24 jam terakhir? Sudah diposting/dijadwalkan hari ini? Kalau ya → SKIP.
4. **Antrean:** item yang lolos masuk antrean dengan status `pending`.
5. **Producer (tiap 5 menit):** ambil item pending pertama (FIFO) → produksi paket seperti slot biasa → posting via Buffer **langsung** (`shareNow`, tanpa `--at`) → ubah status jadi `done` → kirim notifikasi (mis. Telegram).

Prinsip: **lebih baik diam daripada false alarm.** Kualitas > kuantitas.

---

## Alur 7 — Standar Konten

- **Verifikasi 2 sumber** untuk setiap berita. Pakai **"diduga"** bila belum pasti.
- **Footage HD + atribusi** sumber dan akun footage di caption.
- **Patuhi UU ITE** — ekstra hati-hati di slot politik.
- **Anti-duplikat:** baca log hari ini SEBELUM pilih berita.
- **Threads:** maks 500 karakter, maks 5 "link" (hapus TLD).
- **DILARANG:** konten Malaysia, gosip artis, hewan unik, hoaks.
- **JANGAN posting test ke channel live.**

---

## Troubleshooting Cepat

| Gejala | Penyebab & solusi |
|---|---|
| Buffer: "Video could not be read from its URL" | Video belum faststart → mux `-movflags +faststart`, upload ulang |
| Posting TikTok gagal, IG berhasil | Metadata IG ikut ke panggilan TikTok → pisah panggilan, `--ig-reel` hanya untuk IG |
| Caption Threads kepotong / error | Lebih dari 500 karakter atau >5 "link" → pendekkan, hapus TLD |
| Video distorsi / gepeng | Salah aspek saat zoompan → pre-crop ke 1080x1920 dulu |
| ffmpeg error RC 234 | Filter `format` ditulis sebagai chain `;` terpisah → pakai comma-chain `,format=yuv420p[label]` |
| Frame hasil QC aneh / nge-blend | QC pakai `-vf fps=1` menghasilkan frame palsu → pakai `select` by n/timestamp |
| Concat slideshow gagal | SAR segmen beda → tambahkan `setsar=1` per input |
| Caption overlay geser / nge-blend | Chained overlay + `enable` + PNG infinite → pakai pendekatan segmen 8 detik |
| URL video tidak bisa dibuka | Hosting mati / URL salah / directory listing — cek server & tunnel |
| Berita yang sama tayang 2x sehari | Lupa cek log harian → jadikan cek duplikat langkah pertama tiap produksi |

---

## Ringkasnya

1. Buffer + 3 channel + API key → 2. Hosting video (URL publik, listing MATI) → 3. Script posting GraphQL → 4. Produksi konten (riset → verifikasi 2 sumber → footage → caption → faststart → upload) → 5. Cron per slot → 6. (Opsional) Breaking news → 7. Patuhi standar.

_Mulai dari 1 slot manual yang berhasil tayang. Kalau 1 slot jalan, tinggal diduplikasi ke 5 slot + cron._
