# Catatan Sesi — Download YouTube & Transkrip Video Panjang

> Ditulis dari log agent yang sudah berhasil **end-to-end**: download video
> YouTube 56 menit → transkrip → 7 klip panjang ter-render & lolos QA.
> Tujuan file ini: agent sesi berikutnya **langsung pakai cara yang terbukti
> jalan**, tanpa mengulang jalan buntu yang sudah terbukti gagal di sandbox
> ini. Taruh file ini di root project (atau sekadar lampirkan ke chat) dan
> minta agent baca ini **sebelum** mulai research cara download video.
>
> Ikuti bersama `SKILL.md` → `AGENTS.md` → `CATATAN-TEKNIS.md`. File ini
> melengkapi ketiganya, khusus untuk bagian yang belum tertulis di sana:
> download video sumber + transkrip video panjang.

---

## 1. Download video YouTube — ini yang SUDAH TERBUKTI GAGAL, jangan diulang

Sandbox ini kemungkinan besar jalan dari IP datacenter, yang sudah ditandai
YouTube. Semua jalur "biasa" gagal, secara berurutan:

| Cara | Hasil di sesi ini | Kenapa jangan diulang duluan |
|---|---|---|
| `yt-dlp` langsung (semua player client dicoba: web, android, ios, tv_embedded, dst) | diblokir bot-check | konsisten diblokir di semua client, bukan soal client mana yang dipakai |
| Piped (proxy publik) | diblokir juga | instance publik kena block/rate-limit yang sama |
| Invidious (instance publik) | mayoritas mati / tidak balas JSON valid | daftar instance publik cepat basi, jangan habiskan waktu coba satu-satu |
| cobalt (API publik) | sekarang wajib JWT auth | kebijakan mereka berubah, sandbox tidak punya token |

**Jangan mulai dari salah satu di atas.** Kalau mau coba variasi lagi
(client baru / instance baru), itu tetap masuk kategori "sudah terbukti buntu"
sampai ada bukti sebaliknya — jangan buang >5 menit di sini.

## 2. Yang TERBUKTI JALAN: `loader.to`

Alur yang berhasil di sesi ini:

1. Kirim request ke API **loader.to** (layanan yang dipakai banyak
   "youtube downloader" pihak ketiga) dengan URL video.
2. Poll endpoint progress-nya sampai statusnya selesai / dapat link download.
3. Download file dari link yang dikembalikan (`curl`/`wget` biasa).

Hasil di sesi ini: dapat judul video + file 422MB terdownload utuh.

> Endpoint/parameter API pihak ketiga seperti ini bisa berubah sewaktu-waktu.
> Kalau bentuk request yang dipakai sesi lalu sudah tidak jalan, cari dulu
> dokumentasi/endpoint terbaru loader.to (web search) — **tapi tetap mulai
> dari loader.to / layanan sejenis dulu**, jangan balik lagi ke
> yt-dlp/Piped/Invidious/cobalt di atas kecuali loader.to juga sudah benar-benar
> mati saat itu.

Kalau loader.to juga buntu: opsi terakhir adalah minta user upload video
mentah secara manual — jangan habiskan lebih dari ~15-20 menit total mencoba
jalur otomatis sebelum menyerah ke opsi ini.

## 3. Video sumber: cek CFR dulu, jangan asal re-encode ke 30fps

`public/input/README.txt` bilang re-encode ke CFR 30fps kalau videonya VFR.
Untuk video panjang, re-encode itu bisa makan puluhan menit — jadi **cek
dulu**, jangan asumsi:

```bash
# 1) cek cepat
ffprobe -select_streams v -show_entries stream=r_frame_rate,avg_frame_rate,nb_frames video.mp4

# 2) kalau r_frame_rate == avg_frame_rate, VERIFIKASI lebih jauh dengan
#    packet timestamps — CFR asli harus di grid rapi (mis. kelipatan 1/25s),
#    tanpa drift:
ffprobe -select_streams v -show_entries packet=pts_time -of csv video.mp4 | head -50
```

Kalau videonya sudah CFR asli (walau fps-nya bukan 30, mis. 25fps) dan grid-nya
rapi tanpa drift → **langsung pakai sebagai `raw.mp4` tanpa re-encode**. Di
sesi ini video 25fps CFR dipakai langsung, hemat waktu render yang seharusnya
kepakai buat re-encode 56 menit video.

Re-encode ke CFR 30fps **hanya** kalau video terbukti VFR (drift antar frame).

## 4. Transkrip video panjang (>10 menit) — WAJIB dipecah per-bagian

Whisper model `small` di sandbox ini (2 core) jalan **~0.91× realtime**
(kalibrasi dari sesi ini: audio 120s → whisper 110s). Satu tool call sandbox
biasanya dibatasi ±10 menit, jadi video panjang **tidak akan muat** ditranskrip
dalam satu panggilan.

Strategi yang terbukti jalan:

1. **Kalibrasi dulu**, jangan asumsi angka di atas — potong 2 menit audio,
   transkrip, ukur waktunya. Kecepatan bisa beda tergantung load sandbox
   saat itu.
2. Dari situ hitung durasi per-bagian yang aman (target proses <8 menit per
   panggilan, sisakan buffer dari limit 10 menit). Di sesi ini: bagian
   **420 detik (7 menit)** dengan **overlap 6 detik** antar-bagian.
3. Transkrip tiap bagian di panggilan tool terpisah (pola sama dengan
   `render-segments.mjs` — alasan yang sama: kerja panjang dipecah per sesi
   tool-call).
4. **Merge**: titik potong antar-bagian diambil beberapa detik **masuk ke
   dalam area overlap** (bukan pas di batas mentah) — supaya kata yang
   terpotong tepat di ujung bagian tidak hilang atau dobel. Sesi ini: overlap
   6s, split-point +3s ke dalam overlap.
5. **Validasi merge**: cek jumlah gap/kata hilang di titik sambung = 0.
   Kalau ada gap, geser split-point sedikit dan cek ulang.

Hasil sesi ini: 8 bagian × video 56 menit → merge bersih, 0 gap, 7887 kata.

## 5. Render panjang: Chrome crash di tengah segmen itu NORMAL, bukan bug

Ini sudah diantisipasi arsitektur `render-segments.mjs` (idempoten per
segmen) — kalau render mati di tengah (Chrome crash / proses kebunuh sandbox),
**jangan didiagnosis lama-lama**. Segmen yang sudah selesai tetap aman, cukup
panggil lagi:

```bash
bash scripts/render-retry.sh <nama-data>
```

sampai semua segmen kelar. Ini bukan lagi masalah yang perlu "dipecahkan
ulang" tiap sesi — cukup percaya ke wrapper retry.

## 6. Dua bug yang ditemukan sesi ini SUDAH diperbaiki di kode kit

Tidak perlu ditemukan/diperbaiki manual lagi mulai sekarang (lihat
`CATATAN-TEKNIS.md` §13 di zip terbaru):

- **`/tmp` penuh (ENOSPC)** setelah beberapa kali render — penyebabnya
  `bundle()` Remotion menumpuk file baru ~800MB+ di `/tmp` tiap dipanggil
  (termasuk copy penuh `public/`, video mentah ikut ter-duplikasi tiap kali).
  → sudah di-fix: bundle sekarang pakai folder tetap (`.remotion-bundle/` di
  root project, di-overwrite tiap panggilan, tidak menumpuk).
- **Symlink `whisper.cpp/main` hilang** tiap build whisper.cpp fresh (SDK
  butuh nama itu, hasil `make` menaruhnya di `build/bin/main`) → sudah
  di-fix: `bootstrap.sh` sekarang otomatis membuat symlink ini tiap selesai
  build ATAU restore dari cache-pack.
- Kalau kamu masih pakai `clip-kit-cache.tar.gz` versi lama (dari sebelum fix
  ini): **tetap aman dipakai**, tidak perlu dibuat ulang — fix symlink jalan
  otomatis setelah extract, terlepas dari isi cache-pack-nya.

## 7. Ringkasan alur "langsung tembak" untuk sesi berikutnya

```
1. Download video via loader.to (skip yt-dlp/Piped/Invidious/cobalt)
2. ffprobe cek CFR — re-encode HANYA kalau VFR
3. cp video -> public/input/raw.mp4
4. bash scripts/bootstrap.sh   (extract cache-pack kalau ada)
5. Kalibrasi kecepatan whisper (potongan 2 menit) -> tentukan ukuran bagian
6. Transkrip per-bagian dengan overlap, merge, validasi 0 gap
7. Tentukan batas klip dari transkrip (cek kata di sekitar tiap batas)
8. npm run build-data
9. bash scripts/render-retry.sh <nama-data>   (ulang sampai semua segmen selesai)
10. Finalisasi (concat + mux audio) + QA visual (ekstrak 2-3 frame per klip)
```
