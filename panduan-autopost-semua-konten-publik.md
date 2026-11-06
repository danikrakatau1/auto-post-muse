# Sistem Autopost Semua Konten (IG + TikTok + Threads)

_Panduan alur autopost untuk segala jenis konten — berita, fashion, kuliner, hiburan, edukasi, motivasi, dan lainnya. Versi publik, tanpa data internal._

## Prinsip Dasar

Mesinnya **sama untuk semua konten**: riset → produksi video → hosting → Buffer → cron. Yang berubah cuma **isi kontennya** — sumber, tingkat verifikasi, jenis footage, dan gaya caption.

```
[Sumber konten (tergantung jenis)]
        ↓
[Produksi: video vertikal + caption overlay]
        ↓
[Upload video ke hosting → URL publik]
        ↓
[Buffer API] → IG / TikTok / Threads (terjadwal)
        ↓
[Cron] → jalan otomatis tiap slot
```

## Satu Akun Satu Niche

Aturan praktis yang paling penting: **satu akun = satu niche**. Algoritma IG/TikTok/Threads membaca konsistensi topik — akun yang hari ini berita, besok fashion, lusa motivasi akan sulit direkomendasikan ke audiens yang tepat.

Solusinya bukan bikin mesin baru per niche, tapi **satu mesin melayani banyak akun**: tiap akun punya tema sendiri, jadwal slot sendiri, dan log sendiri. Buffer paket gratis maks 3 channel per akun — untuk banyak akun, pakai akun Buffer terpisah per akun sosmed, atau naik ke paket berbayar.

---

## Matriks Jenis Konten

| Jenis konten | Sumber | Verifikasi | Footage | Caption |
|---|---|---|---|---|
| **Berita / viral** | Portal berita, medsos | WAJIB 2 sumber, "diduga" bila ragu | Video viral asli (yt-dlp), atribusi uploader | Hook + fakta + atribusi |
| **Fashion / produk** | Katalog, supplier, tren | Foto produk asli, harga & stok dicek | Foto/video produk (slideshow Ken Burns kalau tidak ada video) | Hook + harga + CTA keranjang |
| **Kuliner** | Tempat makan viral, review | Alamat + harga dicek 1 sumber | Video makanan / suasana tempat | Hook + lokasi + harga + ajakan |
| **Hiburan / lucu** | Medsos, tren | Ringan — pastikan bukan hoaks & bukan konten curian | Video asli + atribusi kreator | Hook lucu + hashtag tren |
| **Edukasi / tips** | Sumber tepercaya (jurnal, ahli, dokumentasi resmi) | 1–2 sumber, cek fakta | Slideshow / rekaman layar / footage relevan | Hook + isi tips + sumber |
| **Motivasi / quotes** | Buku, tokoh, pengalaman | Atribusi quote harus benar | Slideshow teks / footage suasana | Quote + konteks singkat |
| **Olahraga / bola** | Media olahraga, akun resmi klub | 2 sumber untuk skor/transfer | Cuplikan pertandingan / foto | Hook + skor/fakta + hashtag tim |

### Yang berubah per jenis konten
- **Sumber riset** — dari mana konten dicari.
- **Tingkat verifikasi** — berita & olahraga: ketat (2 sumber). Hiburan & motivasi: ringan (cek atribusi & keaslian).
- **Jenis footage** — video viral, foto produk, slideshow, atau rekaman layar.
- **Gaya caption** — hook-nya disesuaikan (fakta vs ajakan beli vs lucu).

### Yang TIDAK berubah
- Buffer API, cara posting, faststart, batas Threads 500 karakter.
- Hosting video + upload + penamaan file `YYYY-MM-DD-HHMM.mp4`.
- Cron per slot, log harian, aturan anti-duplikat.
- Overlay caption (template bisa sama, wording disesuaikan).

---

## Alur 1 — Menyiapkan Buffer

1. Daftar di `buffer.com` (paket gratis: maks 3 channel per akun).
2. Hubungkan **Instagram**, **TikTok**, dan **Threads** lewat menu Channels.
3. Ambil API key: `publish.buffer.com` → **Settings → API** → salin **personal API key** (bukan token developers/apps).
4. Simpan key sebagai environment variable — **jangan tulis di dalam script, jangan commit ke repo publik**:
   ```bash
   echo 'export BUFFER_API_KEY="ISI_KEY_DI_SINI"' >> ~/.bashrc
   source ~/.bashrc
   ```

## Alur 2 — Menyiapkan Hosting Video

Buffer butuh **URL publik** video:

1. Buat folder video di VPS (`mkdir -p ~/videos`).
2. Jalankan HTTP server (nginx disarankan).
3. Buat URL publik ke folder itu — lewat domain sendiri atau tunnel:
   ```
   https://<domain-anda>/2026-11-05-0700.mp4
   ```
4. Tes: upload 1 video, buka URL-nya dari HP — video harus bisa diputar.

> ⚠️ **Matikan directory listing!** Server file default biasanya menampilkan daftar semua file di folder. Siapa pun yang tahu URL hosting bisa melihat dan mengunduh SEMUA video Anda. Di nginx: `autoindex off;`.

## Alur 3 — Posting lewat Buffer API

Buffer memakai **GraphQL API** di `https://api.buffer.com` (selalu POST, header `Authorization: Bearer <API key>`).

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

- `mode: customScheduled` + `dueAt` (UTC) = posting terjadwal.
- `mode: shareNow` = posting langsung.
- `metadata: { instagram: { type: reel } }` = **HANYA untuk Instagram**.

Pemakaian (3 panggilan terpisah — jangan digabung!):

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

## Alur 4 — Produksi Konten (per slot)

Urutan kerja setiap slot (~30–60 menit sebelum jam tayang):

1. **Cari konten** — sesuai jenisnya (lihat matriks di atas). Pilih yang **fresh** dan sesuai tema slot.
2. **Cek duplikat dulu** — buka log harian; JANGAN pakai konten yang sudah dipakai slot lain hari ini.
3. **Verifikasi** — sesuai tingkatnya di matriks. Berita: 2 sumber + "diduga" bila ragu. Produk: harga & stok dicek. Quote: atribusi harus benar.
4. **Siapkan footage** — video asli / foto produk / slideshow / rekaman layar. Video: download pakai `yt-dlp` + atribusi kreator. Foto: slideshow ffmpeg (efek Ken Burns) ukuran 1080x1920.
5. **Caption overlay** — gambar teks (PNG transparan) berisi hook. Maks ~30 karakter per baris. Overlay ke video dengan ffmpeg — cara paling stabil: **pendekatan segmen** (potong video jadi segmen 8 detik, satu overlay sederhana per segmen, lalu gabung dengan `concat`).
6. **Finalisasi & upload** — mux faststart, upload ke hosting (mis. `scp`), catat URL publiknya. Nama file: `YYYY-MM-DD-HHMM.mp4`.
7. **Tulis caption + schedule** — caption santai + hashtag + atribusi/CTA sesuai jenis konten. Threads maks 500 karakter. Schedule via script Alur 3 dengan `--at` jam tayang.
8. **Catat ke log harian** (`posting-YYYY-MM-DD.md`): judul, jam tayang, ID post per channel. Log ini dipakai untuk cek duplikat.

## Alur 5 — Otomatisasi dengan Cron

| Cron jalan | Slot tayang | Isi kerja |
|---|---|---|
| 06:00 | — | Brief pagi: riset topik hari itu |
| 06:30 | 07:00 | Produksi + schedule slot 1 |
| 10:30 | 11:00 | Produksi + schedule slot 2 |
| 14:51 | 15:00 | Produksi + schedule slot 3 |
| 18:51 | 19:00 | Produksi + schedule slot 4 |
| 21:30 | 22:00 | Produksi + schedule slot 5 |
| tiap 20 mnt | — | Monitor tren/breaking (opsional) |
| tiap 5 mnt | — | Produksi + tayang antrean yang pending |
| 23:00 | — | Rekap harian |

```cron
# Brief pagi + slot harian (WIB)
0 6 * * * /usr/bin/python3 /home/user/autopost/produksi.py --brief
30 6 * * * /usr/bin/python3 /home/user/autopost/produksi.py --slot 07:00
30 10 * * * /usr/bin/python3 /home/user/autopost/produksi.py --slot 11:00
51 14 * * * /usr/bin/python3 /home/user/autopost/produksi.py --slot 15:00
51 18 * * * /usr/bin/python3 /home/user/autopost/produksi.py --slot 19:00
30 21 * * * /usr/bin/python3 /home/user/autopost/produksi.py --slot 22:00

# Monitor + producer antrean
*/20 * * * * /usr/bin/python3 /home/user/autopost/monitor.py
*/5 * * * * /usr/bin/python3 /home/user/autopost/producer.py
```

Satu script `produksi.py` bisa menangani semua slot dan semua jenis konten — tinggal ganti **mode konten** per slot (mis. slot pagi = berita, slot siang = kuliner, dst.) atau satu akun satu mode.

Awalnya cron bisa memanggil script yang **hanya menyiapkan paket + minta persetujuan Anda**; setelah yakin, ubah ke full otomatis.

## Alur 6 — Tren / Breaking (opsional, untuk konten yang sensitif waktu)

Untuk jenis konten yang sensitif waktu (berita, bola, tren viral):

1. **Monitor (tiap 20 menit):** cari topik via pencarian berita + pencarian sosial.
2. **Filter ketat** — proses HANYA kalau semua lolos: kejadian dalam 3 jam terakhir, skala besar / viral nasional, terverifikasi (berita: 2 sumber), bukan hoaks / konten curian.
3. **Cek duplikat:** sudah pernah diproses 24 jam terakhir? Sudah diposting hari ini? Kalau ya → SKIP.
4. **Antrean:** yang lolos masuk antrean (`pending`).
5. **Producer (tiap 5 menit):** ambil antrean pertama (FIFO) → produksi → posting via Buffer **langsung** (`shareNow`) → status `done` → kirim notifikasi.

Untuk konten yang tidak sensitif waktu (motivasi, edukasi, produk), alur ini bisa di-skip — cukup slot harian.

## Standar Umum Semua Konten

- **Verifikasi sesuai jenisnya** (lihat matriks). Jangan menyebar hoaks.
- **Footage + atribusi** — cantumkan sumber/kreator di caption.
- **Patuhi UU ITE** — ekstra hati-hati di konten politik & sensitif.
- **Anti-duplikat:** baca log hari ini SEBELUM pilih konten.
- **Threads:** maks 500 karakter, maks 5 "link" (hapus TLD).
- **JANGAN posting test ke channel live.**

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
| Konten yang sama tayang 2x sehari | Lupa cek log harian → jadikan cek duplikat langkah pertama tiap produksi |

---

## Ringkasnya

1. Buffer + 3 channel + API key → 2. Hosting video (URL publik, listing MATI) → 3. Script posting GraphQL → 4. Produksi konten (riset → verifikasi → footage → caption → faststart → upload) → 5. Cron per slot → 6. (Opsional) Monitor tren/breaking → 7. Patuhi standar.

**Satu mesin, semua jenis konten.** Yang beda cuma isi kontennya — dan satu akun sebaiknya satu niche.

_Mulai dari 1 slot manual yang berhasil tayang. Kalau 1 slot jalan, tinggal diduplikasi ke 5 slot + cron, lalu tambah akun/niche lain di mesin yang sama._
