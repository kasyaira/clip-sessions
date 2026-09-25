# Catatan Sesi — Download YouTube & Transkrip Video Panjang

> Ditulis dari log agent yang sudah berhasil **end-to-end**: download video
> YouTube 56 menit → transkrip → 7 klip panjang ter-render & lolos QA.
> Di-update lagi setelah **sesi-08** (video 13 menit, 8 klip, upload per-klip
> ke GitHub) sukses end-to-end dengan metode yang sama + beberapa perbaikan.
> Tujuan file ini: agent sesi berikutnya **langsung pakai cara yang terbukti
> jalan**, tanpa mengulang jalan buntu yang sudah terbukti gagal di sandbox
> ini. Taruh file ini di root project (atau sekadar lampirkan ke chat) dan
> minta agent baca ini **sebelum** mulai research cara download video.

---

## 1. Download video YouTube — ini yang SUDAH TERBUKTI GAGAL, jangan diulang

Sandbox ini kemungkinan besar jalan dari IP datacenter, yang sudah ditandai
YouTube. Semua jalur "biasa" gagal, secara berurutan:

| Cara | Hasil di sesi ini | Kenapa jangan diulang duluan |
|---|---|---|
| `yt-dlp` langsung (semua player client dicoba: web, android, ios, tv_embedded, dst) | diblokir bot-check | konsisten diblokir di semua client, bukan soal client mana yang dipakai |
| Piped (proxy publik) | diblokir juga | instance publik kena block/rate-limit yang sama |
| Invidious (instance publik) | mayoritas mati / tidak balas JSON valid | daftar instance publik cepat basi, jangan habiskan waktu coba satu-satu |
| cobalt (API publik) | wajib JWT auth / tunnel URL balik 0 byte | kebijakan mereka berubah, sandbox tidak punya token |

**Jangan mulai dari salah satu di atas.** Kalau mau coba variasi lagi
(client baru / instance baru), itu tetap masuk kategori "sudah terbukti buntu"
sampai ada bukti sebaliknya — jangan buang >5 menit di sini.

## 2. Yang TERBUKTI JALAN (2x, termasuk sesi-08): `loader.to`

Alur yang berhasil, **dengan endpoint persisnya**:

```bash
# 1) kirim request (format=720 cukup; 1080 juga ada)
curl -s "https://loader.to/ajax/download.php?format=720&url=<URL-YOUTUBE-ENCODED>" \
  -H "User-Agent: Mozilla/5.0"
# -> {"success":true,"id":"v2_stream_xxx","progress_url":"https://lto2.affadaffa.com/api/progress?id=..."}

# 2) poll sampai "progress":1000 dan dapat "download_url"
curl -s "https://lto2.affadaffa.com/api/progress?id=v2_stream_xxx" -H "User-Agent: Mozilla/5.0"

# 3) download file dari download_url (curl -sL, jangan lupa -L)
```

Hasil sesi-08: video 13 menit 67MB 720p utuh dalam <10 menit (server memproses
sementara kita bootstrap environment — **jalankan paralel dengan bootstrap.sh**).

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
#    packet timestamps — CFR asli harus SEMUA PTS jatuh di grid fps:
ffprobe -select_streams v -show_entries packet=pts_time -of csv video.mp4 | head -4000
# hitung: pts * fps harus bulat semua (toleransi 0.02). Delta antar paket bisa
# aneh-aneh (B-frame reorder) — yang penting semua nilai PTS on-grid.
```

Kalau videonya sudah CFR asli (walau fps-nya bukan 30, mis. 23.976fps) →
**langsung pakai sebagai `raw.mp4` tanpa re-encode**. Remotion menangani
konversi fps saat render. Di sesi-05 (25fps) dan sesi-08 (23.976fps) video
dipakai langsung — hemat puluhan menit.

## 4. Transkrip video panjang (>8-10 menit) — WAJIB dipecah per-bagian

Whisper model `small` di sandbox 2-core kecepatannya bervariasi per sesi
(sesi-05: ~0.91x realtime; sesi-08: **1.12x realtime**) — **selalu kalibrasi
dulu** dengan slice 120s sebelum menentukan ukuran bagian. Satu tool call
biasanya dibatasi ±10 menit.

Mulai **sesi-08** ada script siap pakai `scripts/transcribe-parts.mjs`
(menggantikan prosedur manual):

```bash
node scripts/transcribe-parts.mjs calib             # ukur kecepatan + rekomendasi durasi bagian
node scripts/transcribe-parts.mjs part 1 0 396      # bagian 1: mulai 0s, durasi 396s
node scripts/transcribe-parts.mjs part 2 390 382.2  # bagian 2: mulai 390s (overlap 6s)
node scripts/transcribe-parts.mjs merge raw-id 393  # gabung, split-point +3s ke dalam overlap
```

Merge otomatis + validasi 0 gap. **Dua artefak whisper yang DITEMUKAN sesi-08
dan SUDAH di-fix di script merge** (jangan diagnosa ulang kalau ketemu lagi):

- **Blok kata timestamp terbalik / ter-halusinasi**: whisper (max-len 1 + DTW)
  kadang meng-emit satu kalimat dengan timestamp MENURUN dan nilai yang
  didaur ulang dari konteks sebelumnya (kata-kata itu menimpa rentang waktu
  yang sudah dipakai kalimat lain). JANGAN sekadar dibalik urutannya —
  solusinya: pertahankan urutan teks, lalu **re-time seluruh blok proporsional
  di antara kata sehat sebelum/sesudahnya** (sudah otomatis di merge).
- **Kata zero-duration** (start == end, ~12% dari semua kata di sesi-08):
  diberi durasi minimal otomatis — tanpa ini kata-kata terbuang oleh filter
  `build-clips` (`end-start > 0.015`).

## 5. Render panjang: Chrome crash di tengah segmen itu NORMAL, bukan bug

Ini sudah diantisipasi arsitektur `render-segments.mjs` (idempoten per
segmen) — kalau render mati di tengah (Chrome crash / tool call timeout /
proses kebunuh sandbox), **jangan didiagnosis lama-lama**. Segmen yang sudah
selesai tetap aman, cukup panggil lagi:

```bash
node scripts/render-segments.mjs <nama-klip>.json   # ulang sampai "semua pending selesai"
```

Sesi-08: klip 133-155 detik (3 segmen) butuh 2 panggilan per klip — itu
normal, bukan error. **Fix penting sesi-08**: folder segmen sekarang
PER-KLIP (`out/segments/<nama-klip>/`) — versi lama pakai `out/segments/`
bersama, akibatnya klip kedua menganggap seg-00000.mp4 milik klip pertama
sudah "selesai" dan hasilnya rusak.

Setelah semua segmen klip selesai, gabung + mux audio:

```bash
node scripts/finalize-clip.mjs <nama-klip>   # concat segmen + audio sumber -> out/clips/<nama>.mp4
```

Audio selalu diambil utuh dari video sumber (`ffmpeg -ss <start> -t <dur>`),
bukan dari render (yang muted) — bebas glitch di batas segmen.

## 6. Bug lama yang SUDAH diperbaiki di kode kit (revisi sesi-08)

- **`/tmp` penuh (ENOSPC)** setelah beberapa kali render — `bundle()` Remotion
  menumpuk file besar di /tmp tiap dipanggil. → sudah di-fix di
  `render-segments.mjs` + `render-all.mjs`: bundle sekarang pakai folder tetap
  `.remotion-bundle/` di root project (parameter `outDir`).
- **Symlink `whisper.cpp/main` hilang** tiap build whisper.cpp fresh (SDK
  butuh nama itu, hasil `make` menaruhnya di `build/bin/main`) → kalau
  `installWhisperCpp` rewel "executable missing", cukup:
  `ln -sf build/bin/main whisper.cpp/main`.
- **`tsconfig.json` hilang dari zip project** → gejala: `npx remotion browser
  ensure` GAGAL dengan pesan "Could not find a tsconfig.json" (di sesi-08
  sempat salah didiagnosis sebagai masalah versi zod). Fix: buat tsconfig.json
  standar (ada di zip revisi sekarang).
- **CLI `remotion still`**: flag input props di v4.0.526 adalah `--props`,
  BUKAN `--input-props` (yang diam-diam diabaikan tanpa error!).

## 7. Upgrade gaya subtitle sesi-08 — karaoke per-kata (kuning)

Permintaan user: "text pada saat popup warnanya harus keliatan, sesuai irama
bicara per kata, misal kuning saat kata disebut". Implementasi (sudah di
`src/config.ts` + `src/components/Subtitles.tsx`):

- Tiap kata punya state dari timing whisper word-level: **kata yang sedang
  diucap** → blend putih→kuning `#FFDE1A` dalam 0.09s + pop 1.08x;
  **kata yang sudah diucap** → tetap kuning (progressive); **kata berikutnya**
  → putih meredup (opacity 0.75).
- Keterbacaan: strokeWidth 13→15, shadow dikuatkan, warna dasar tetap putih
  dengan outline gelap `#151310`.
- Semua parameter di `SUBTITLE.activeWord` (config) — bisa diubah tanpa
  sentuh komponen.
- QA cepat tanpa render penuh: `npx remotion still src/index.ts VerticalClip
  out.png --frame=<N> --props='<json>'` lalu cek frame pakai VLM.

## 8. Upload per-klip ke GitHub (workflow sesi-08)

Permintaan user: "setelah selesai 1 clip langsung upload ke git, satu persatu
tanpa menunggu semuanya selesai". Pola yang terbukti jalan:

```
per klip: render-segments (ulang sampai tuntas) -> finalize-clip -> UPLOAD -> klip berikutnya
```

- Repo: `kasyaira/clip-sessions`, struktur `sesi-XX-slug/clip-NN-nama.mp4`
  + `metadata.json` per sesi (lihat sesi yang sudah ada untuk format).
- Upload via GitHub Contents API PUT (base64), payload ditulis ke file dulu
  (file 30MB jadi ~40MB base64 — terlalu besar untuk argumen CLI).
- Script siap pakai: `/home/z/my-project/scripts/gh-upload.sh <file> <path>`.
- Kalau balas "Repository rule violations found / Timed out validating rule"
  → **transien**, langsung retry saja (sesi-08: 1x gagal lalu sukses di
  percobaan kedua).

## 9. Ringkasan alur "langsung tembak" untuk sesi berikutnya

```
1. Download video via loader.to (skip yt-dlp/Piped/Invidious/cobalt)
   — jalankan PARALEL dengan bootstrap.sh
2. bash scripts/bootstrap.sh   (buat tsconfig.json kalau zip tidak menyertakan)
3. ffprobe cek CFR — re-encode HANYA kalau VFR (on-grid check, lihat §3)
4. cp video -> public/input/raw.mp4
5. node scripts/transcribe-parts.mjs calib  -> tentukan ukuran bagian
6. transcribe per-bagian dengan overlap 6s -> merge -> 0 masalah sambungan
7. Tentukan batas klip dari transkrip (cut di gap antar-kalimat, presisi
   per kata; klip 60-155s, topik harus komplet sampai kalimat penutup)
8. npm run build-data
9. PER KLIP: node scripts/render-segments.mjs <klip>.json (ulang sampai tuntas)
            node scripts/finalize-clip.mjs <klip>
            bash gh-upload.sh out/clips/<klip>.mp4 sesi-XX-slug/clip-NN-<klip>.mp4
10. Upload metadata.json sesi + CARA-KERJA.md ter-update + zip kit revisi
11. QA visual: ekstrak 2-3 frame per klip -> cek via VLM
    (ingat: --frame pakai waktu RELATIF klip, bukan timestamp video sumber)
```

Catatan batas klip (pelajaran sesi-08): potongan terbaik ada di **gap antar
kalimat**; kalau kata terakhir dan kata pertama klip berikutnya menempel
persis (zero-gap), set `end` sedikit SEBELUM kata pertama klip berikutnya
(kata itu tetap terdengar di audio karena tail 0.45s, tapi tidak jadi
subtitle menggantung). `lead` 0.2s memberi napas di awal klip yang dipotong
dari tengah video.
