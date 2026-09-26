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

---

# ADDENDUM SESI-09 (video 51.5 menit "Bobon Mau Bantu Lebih Banyak Orang Lewat Politik")

> Sesi ini menjalankan LAGI alur §7 end-to-end dan menemukan beberapa jalan
> buntu baru + fix-nya. Semua di bawah SUDAH DIVERIFIKASI JALAN di sesi ini.
> Script baru semuanya sudah masuk zip kit terbaru (lihat daftar §9).

## 8. Metode download yang BERHASIL (konfirmasi ulang §2): loader.to

Skrip siap pakai sekarang ada di kit: `scripts/loader-download.mjs`

```
node scripts/loader-download.mjs "https://youtu.be/<id>" /path/out.mp4 720
```

- Alur: POST `loader.to/ajax/download.php?format=720&url=...` → dapat `id` →
  poll `loader.to/ajax/progress.php?id=<id>` tiap 3s sampai ada `download_url`
  → fetch file-nya (stream ke `.part`, rename setelah utuh).
- Hasil sesi ini: video 51:29 (3089s), 1280x720 **CFR 30fps asli** (r_frame_rate
  == avg_frame_rate == 30/1, semua pts di grid 1/30) → dipakai langsung
  sebagai `raw.mp4` TANPA re-encode (§3).
- Catatan: tool call bisa ke-timeout sebelum skrip selesai — file `.part`
  yang belum direname berarti belum utuh; file yang SUDAH direname tapi proses
  terbunuh TETAP VALID (cukup `ffprobe` untuk verifikasi). Kalau belum selesai,
  jalankan lagi skrip yang sama (tidak otomatis resume — mulai dari progress
  terakhir server biasanya cepat, atau ulang total).
- oEmbed (`https://www.youtube.com/oembed?url=...&format=json`) selalu jalan
  untuk konfirmasi judul/channel DULU sebelum download.

## 9. Script baru di kit (semua terverifikasi sesi ini)

| Script | Fungsi |
|---|---|
| `scripts/transcribe-parts.mjs` | transkrip per-bagian (§4) jadi 4 subcommand: `calibrate` (ukur kecepatan whisper pakai potongan 120s), `init --part-len 420 --overlap 6` (potong wav per bagian), `part N` (transkrip bagian N, idempoten, cache per bagian), `merge` (gabung + validasi monotonic). Merge rule: bagian menyimpan kata lokal di `[overlap/2, partLen-overlap/2)` — TERBUKTI 0 gap, 0 back-jump. |
| `scripts/finalize-clip.mjs` | concat segmen muted + mux audio asli dari video sumber (rentang clipStart-lead sampai +tail) → `out/clips/<id>.mp4` +faststart. Sekaligus validasi jumlah frame vs target. |
| `scripts/gh-upload.mjs` | upload 1 file ke repo GitHub `clip-sessions` via Contents API; kalau 422/502 otomatis fallback ke Git Blobs/Tree/Commit API. |
| `scripts/gh-push-big.sh` | file BESAR (>±45MB) yang ditolak API: git plumbing (fetch blob-less → hash-object → read-tree → update-index → commit-tree → push). |
| `scripts/qa-frames.mjs` | render frame PNG tertentu dari data klip untuk QA visual cepat. |
| `scripts/words-around.mjs` | lihat kata + timestamp di sekitar detik tertentu — untuk pilih batas klip presisi di batas kalimat. |
| `scripts/dump-transcript.mjs` | dump transkrip per jendela 20s untuk baca struktur topik. |
| `scripts/loader-download.mjs` | download via loader.to (§8). |

## 10. Bug & jalan buntu BARU yang ditemukan sesi ini (beserta fix)

1. **`tsconfig.json` tidak ada di zip kit** → `npx remotion browser ensure`
   GAGAL dengan pesan "Could not find a tsconfig.json". Fix: buat tsconfig
   standar Remotion (sudah ikut zip terbaru).
2. **pip PEP 668** (`externally-managed-environment`) → `pip install cmake`
   ditolak. Fix yang berhasil: `python3 -m venv ~/.venv` lalu
   `~/.venv/bin/python -m pip install cmake` (CATATAN: binary `pip` tidak ada
   di venv ini — WAJIB pakai `python -m pip`). Bootstrap lalu otomatis nemu
   cmake di `~/.venv/bin`.
3. **`@remotion/studio` hilang dari node_modules** padahal dibutuhkan
   `@remotion/bundler` (error `renderEntry.js` MODULE_NOT_FOUND) — paket ini
   TIDAK ada di package.json kit. Fix: `npm install @remotion/studio@4.0.526`
   (WAJIB pin versi sama persis dengan remotion lain). Kalau npm bilang
   "up to date" tapi filenya rusak (sisa install yang terbunuh), hapus folder
   `node_modules/@remotion/studio` dulu baru install ulang.
4. **Kebocoran /tmp LEBIH PARAH dari §6**: bukan cuma `bundle()` yang numpuk
   (`/tmp/remotion-webpack-bundle-*` ±420MB/panggilan), `renderMedia` juga
   meninggalkan `/tmp/remotion-v*-assets*` (±400MB/proses, isinya copy
   `public/` = video mentah). Render beberapa segmen bisa habiskan disk 10GB
   dalam <30 menit → node mati diam-diam (log KOSONG, exit instan = cek
   `df -h` dulu!). Fix: `render-retry.sh` sekarang membersihkan kedua pola
   itu sebelum & sesudah tiap percobaan.
5. **Symlink whisper.cpp/main** (§6) memang belum ada di zip lama — sudah
   ditambahkan ke `bootstrap.sh` (auto `ln -sfn build/bin/main main`).
6. **`ffprobe -of csv=p=0`** mengeluarkan trailing comma ("1584,") →
   `Number()` langsung jadi NaN. Selalu parse dengan regex `/\d+/`.
7. **Frame QA tepat di detik bulat** sering jatuh pas frame pertama chunk
   (animasi pop-in mulai dari opasitas 0) → subtitle kelihatan "hilang".
   Bukan bug — sampling QA di offset .5-.7 detik.
8. **Proses background (`nohup ... &`) tetap dibunuh** saat tool call
   berakhir (konfirmasi § CATATAN-TEKNIS) — bootstrap WAJIB foreground,
   ulang sampai selesai (idempoten per langkah).

## 11. Upload GitHub: batas ukuran & metode yang jalan (TERVERIFIKASI)

- Repo target: `kasyaira/clip-sessions`, 1 folder per sesi
  (`sesi-XX-slug/`) + `metadata.json` per sesi (ikuti format sesi-08).
- **Contents API PUT** oke untuk ± file <±45MB. Mendekati batas sering
  balas **502/504** (transient — boleh retry sekali) atau **422** "file is
  too large to be processed" (permanent untuk ukuran itu).
- **Git Blobs API** juga menolak payload base64 yang sama besar (422
  "input was too large").
- **Metode yang JALAN untuk file besar (terpakai untuk 46-54MB)**:
  `scripts/gh-push-big.sh` = git plumbing:
  ```
  git init work; git remote add origin https://<user>:<PAT>@github.com/...
  git fetch --depth 1 --filter=blob:none origin main   # cuma commit+tree, kecil
  BLOB=$(git hash-object -w file.mp4)
  git read-tree origin/main^{tree}
  git update-index --add --cacheinfo 100644,$BLOB,path/di/repo.mp4
  git commit-tree $(git write-tree) -p origin/main -m "msg"
  git push origin <commit>:main
  ```
  PERINGATAN: proses `push` melakukan lazy-fetch SEMUA blob repo (repo ini
  ±1.4GB!) → work dir membengkak. **Hapus work dir gh-push setelah tiap
  push** (fetch ulang berikutnya cuma beberapa detik).
- Upload per-klip SEGERA setelah klip selesai (workflow user: satu-satu,
  tanpa nunggu batch) — urutan render pakai prioritas topik terkuat dulu
  supaya kalau sesi mati di tengah, yang sudah ter-upload adalah yang paling
  penting.

## 12. Word-level yellow highlight (upgrade subtitle sesi-09)

`src/components/Subtitles.tsx` + `src/config.ts` (nilai di
`SUBTITLE.activeWord`):
- Kata yang SEDANG diucap → warna **kuning #FFD60A**, scale 1.08x, glow
  drop-shadow lembut, opasitas penuh; ramp naik 2 frame, tahan 0.3s
  setelah kata selesai, fade balik 3 frame.
- Kata lain → putih, opasitas 0.85 (masih jelas), outline hitam 13px.
- Style single-active (hanya kata aktif yang kuning) — beda dari sesi-08
  yang progresif (kata yang sudah diucap tetap kuning). Keduvalidasi QA
  visual via VLM: highlight presisi mengikuti timestamp kata.
- Konfigurasi semua di `config.ts` (single source of truth) — ganti warna/
  timing di situ tanpa sentuh komponen.

## 13. Alur "langsung tembak" versi sesi-09 (menggantikan §7)

```
 1. oEmbed konfirmasi video → loader-download.mjs (skip yt-dlp/Piped/Invidious/cobalt)
 2. ffprobe CFR → re-encode HANYA kalau VFR (§3)
 3. cp video -> public/input/raw.mp4
 4. bash scripts/bootstrap.sh  (foreground, ulang sampai selesai;
    cek df -h kalau langkah mati diam-diam → bersihkan /tmp/remotion-*)
 5. transcribe-parts.mjs calibrate → init --part-len <hasil kalibrasi> →
    part 0..N (satu tool call per bagian) → merge (harus mono, 0 gap)
 6. dump-transcript.mjs + words-around.mjs → tentukan batas klip di batas
    KALIMAT (bukan asal detik), topik harus tuntas, 111-236s sesi ini
 7. render-inputs/clips.json → npm run build-data
 8. PER KLIP: rm -rf out/segments; render-retry.sh <id> (ulang sampai semua
    segmen); finalize-clip.mjs <id>; QA 1-2 frame (qa-frames / ffmpeg -ss);
    UPLOAD LANGSUNG (gh-upload.mjs / gh-push-big.sh) — jangan nunggu batch
 9. metadata.json per sesi (format sesi-08) → upload
10. Upload zip kit revisi + CARA-KERJA.md terbaru ke repo
```

---

# ADDENDUM SESI-10 (video 13:02 "Menolak Pakar Termasuk NPD Nggak Sih?" — Felix Siauw)

> Video pendek: alur §13 jalan tanpa modifikasi (cuma 2 bagian transkrip).
> TAPI sesi ini menemukan insiden penting soal TOKEN yang wajib diketahui
> semua sesi berikutnya — lihat §14.

## 14. INSIDEN TOKEN: hardcode di zip kit = AUTO-REVOKE oleh GitHub

**Apa yang terjadi:** token PAT yang di-hardcode di `gh-upload.mjs` +
`gh-push-big.sh` (per permintaan "simpan hardcode gapapa") IKUT TER-PUBLISH
ke repo PUBLIK `kasyaira/clip-sessions` lewat `ofc-clip-kit.zip`. GitHub
punya **secret scanning** yang otomatis mendeteksi fine-grained PAT yang
bocor di repo publik dan **langsung me-revoke-nya**. Akibatnya di sesi-10:
- REST API `api.github.com` → 401 "Bad credentials"
- `git push` → "Invalid username or token" (padahal fetch/ls-remote SUKSES
  — MENIPU, karena repo-nya publik: fetch jalan anonim tanpa auth sama
  sekali! Jangan pakai ls-remote/fetch sebagai bukti token valid.)

**Aturan mulai sesi-10 (SUDAH DITERAPKAN DI KIT):**
1. TOKEN TIDAK PERNAH LAGI DITULIS DI FILE KIT. `gh-upload.mjs` dan
   `gh-push-big.sh` sekarang membaca token dari file eksternal DI LUAR
   kit: `/home/z/my-project/work/.ghtoken` (satu baris, isi PAT saja).
2. Sebelum zip kit di-upload ke repo, SELALU scan dulu:
   `rg -l "github_pat_" scripts/ src/ *.md *.json` → harus kosong.
3. Kalau token baru diberikan user lewat chat: tulis ke `.ghtoken`,
   jangan pernah echo nilai token ke file yang akan di-upload.
4. Cara verifikasi token sebelum pakai: `git push --dry-run` (bukan
   ls-remote — lihat poin menipu di atas).

## 15. Catatan alur sesi-10 (video pendek 13 menit)

- Video 782s CFR 23.976fps → langsung `raw.mp4` tanpa re-encode (§3).
- Kalibrasi whisper: 0.96x realtime → part-len 460 → cuma 2 bagian
  (init otomatis hitung), merge 1713 kata 0 gap.
- 5 klip membagi HABIS seluruh video (0-782s, tanpa gap antar klip),
  batas potong di batas kalimat persis via words-around (§13 langkah 6):
  0-220.6 / 220.6-422.6 / 422.6-561.4 / 561.8-661.4 / 661.9-782.0
- Urutan render prioritas topik terkuat dulu (clip-02 ceklist NPD →
  clip-01 mitos → clip-03 → clip-04 → clip-05) — sesuai §11.
- QA piksel cepat (pengganti VLM kalau tidak sempat): ekstrak frame,
  cek 1080x1920 + ada piksel kuning #FFD60A di area subtitle dengan PIL
  — terbukti cukup untuk memastikan word-highlight jalan.
- gh-push-big.sh sekarang otomatis hapus work dir gh-push setelah tiap
  push (§11) — tidak perlu manual lagi.

---

# ADDENDUM SESI-11 (penyelesaian sesi-10: upload 5 klip Felix NPD)

> User kasih token PAT baru (lama di-revoke, lihat §14) + minta clip
> youtu.be/3SkVPuJnGBI. oEmbed judulnya ternyata = video yang SAMA dengan
> sesi-10 — dan hasil kerja sesi-10 MASIH ADA (lihat §16). Sesi ini tidak
> download/transkrip/render ulang sama sekali, cukup verifikasi + upload.

## 16. PENTING: folder upload/ SELAMAT dari reset sandbox

Direktori `/home/z/my-project/upload/` adalah mount jaringan (ossfs) yang
**TIDAK ikut ter-reset** bersama sandbox. Di sesi-11 ditemukan `upload/ofc-clip-kit/`
masih berisi DIRECTORI KERJA SESI-10 UTUH 1.9GB: whisper.cpp ter-build + model
small 487MB + node_modules + chrome headless shell + out/ (5 klip final, transcript
1713 kata, out/data) + cache-pack 54MB.

**Langkah pertama setiap sesi baru: CEK dulu `upload/` sebelum kerja apapun.**
Kalau ada `upload/ofc-clip-kit/` dari sesi lalu, pulihkan dengan rsync (§17)
daripada build dari nol / render ulang.

Konfirmasi video = video sesi lama: oEmbed ID-nya, cocokkan judulnya dengan
metadata render log / CARA-KERJA addendum sebelumnya.

## 17. Copy dari upload/ KE root project: pakai RSYNC, jangan mv

`mv upload/ofc-clip-kit ofc-clip-kit` = COPY antar-mount (~4.4MB/s), tool call
ke-timeout di 120s dan menyisakan copy setengah jadi. Yang benar:

```
rsync -a --exclude='out/' --exclude='clip-kit-cache.tar.gz' \
  --exclude='out-render-*.log' --exclude='out-seg*.log' --exclude='.git' \
  upload/ofc-clip-kit/ ofc-clip-kit/
```

- rsync lanjut otomatis dari file yang sudah tersalin (aman dipanggil ulang).
- `out/` dibiarkan di upload/ — klip final & transcript bisa dibaca/di-upload
  langsung dari path upload/ (gh-upload.mjs terima path absolut).
- Setelah rsync: tulis token baru ke `work/.ghtoken`, lalu `bootstrap.sh`
  (langsung skip semua karena node_modules + whisper + chrome sudah ada).

## 18. Verifikasi ulang klip warisan sebelum upload (sesi-11)

Sebelum upload klip hasil sesi lama, cek cepat (semua lolos di sesi-11):
1. ffprobe tiap klip: 1080x1920, 30fps, stream audio aac ada, durasi masuk akal.
2. QA piksel (§15): ffmpeg ekstrak 1 frame per klip + PIL hitung piksel kuning
   #FFD60A di area subtitle — >50 piksel = word-highlight jalan.
3. Transcript warisan: cek jumlah kata + timestamp akhir ≈ durasi video.

## 19. Metode upload terkonfirmasi ulang (sesi-11)

- 33.4MB / 25.5MB / 27.8MB → Contents API PUT langsung OK.
- 47.4MB / 49.6MB → Contents API 422 → Git Blobs API 422 → `gh-push-big.sh`
  SUKSES (cepat, beberapa puluh detik saja — catatan lazy-fetch 1.4GB di §11
  tidak terjadi lagi di sesi-11).
- Upload klip satu-satu tetap urut topik terkuat dulu (§11).
- README.md repo perlu dirapikan tabel sesinya (belum di-update sejak sesi-05;
  sesi-11 sudah lengkapi s/d sesi-10 — jaga tetap ter-update tiap sesi).

---

# ADDENDUM SESI-12 (video sesi-11: "Kenapa orang-orang pada ngomongin Mas Wapres?" — Ray Restu Fauzi)

> Catatan penomoran: addendum ini labelnya SESI-12 karena addendum sesi
> agent sebelumnya sudah memakai nama "SESI-11" (sesi agent yang hanya
> menyelesaikan upload sesi-10). Untuk FOLDER VIDEO di repo, video ini
> tetap terdaftar sebagai sesi-11 (penomoran mengikuti video, bukan
> sesi agent). Video 31:44, 1280x720, 60fps CFR ASLI (full-scan nol gap),
> dipakai langsung tanpa re-encode → 11 klip full-coverage 0-1904.5s.

## 20. Catatan operasional sesi ini (semua terverifikasi jalan)

1. **Sandbox reset LAGI** — pola pemulihan §16-17 terbukti: `upload/`
   selamat, rsync dari `upload/ofc-clip-kit/` (exclude out/, cache, .git,
   CARA-KERJA.md supaya versi repo tidak tertimpa versi lama), lalu
   `work/.ghtoken` ditulis ulang, bootstrap skip semua.
2. **JANGAN paralel rsync + loader-download** di dua tool call bersamaan —
   bandwidth berbagi, keduanya timeout. Jalankan berurutan.
3. **loader-download bisa "timeout" padahal SUDAH selesai** — tool call
   ke-timeout tepat setelah file ter-rename dari .part. SELALU cek
   `work/raw-new.mp4` ada + ffprobe valid SEBELUM retry download.
4. Sumber 60fps (pertama kali di-project ini) diproses normal — kit render
   30fps, video card 60fps tidak masalah, A/V sinkron (audio di-mux dari
   sumber, bukan hasil render).
5. Kalibrasi whisper 0.99x realtime → part-len 460 → 5 bagian → merge
   4147 kata, mono, 0 back-jump.
6. QA piksel: frame gagal kuning=0 di clip-05 @80.5s ternyata micro-pause
   antar chunk (kata "datang" selesai 840.40, "mulai" mulai 840.43) —
   frame tetangga (±1-2s) semua OK. Sebelum diagnosa gagal, SELALU cek
   3-4 frame tetangga + cek transkrip apakah sedang jeda hening.
7. Upload: 21-22MB → Contents API OK; 28-55MB → langsung gh-push-big.sh
   (semua sukses sekali jalan, cepat — jangan buang waktu coba Contents
   API untuk file >28MB).
8. 11 klip × render ≈ 2-3 panggilan render-retry per klip (masing-masing
   kena timeout 10 menit lalu lanjut) — total ~4 jam untuk video 31 menit.
   Pola: `rm -rf out/segments && render-retry.sh <id> 3` ulang sampai
   EXIT=0, lalu finalize + qa_clip.py + upload SEBELUH klip berikutnya.
9. README repo + metadata.json + CARA-KERJA.md + zip kit di-update tiap
   akhir sesi (tabel sesi di README jangan sampai tertinggal lagi).

## 21. Cache-pack LENGKAP (dengan model) tersimpan permanen di upload/

Bootstrap sesi ini membuat cache-pack baru 526MB yang SUDAH TERMASUK
model whisper small (yang lama 54MB tidak). Salinan sudah ditaruh di
mount persisten: `upload/clip-kit-cache.tar.gz`.

Sesi baru (setelah reset) punya 2 jalur pemulihan — pilih salah satu:
- **Jalur A (paling cepat)**: rsync dari `upload/ofc-clip-kit/` (§17) —
  dapat whisper+model+node_modules+chrome sekaligus (~5 menit).
- **Jalur B**: extract `upload/clip-kit-cache.tar.gz` ke root kit
  (`tar xzf` di dalam folder kit) untuk whisper+model+chrome, lalu
  `npm ci` untuk node_modules (~2-3 menit).
