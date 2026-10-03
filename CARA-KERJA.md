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

---

# ADDENDUM SESI-13 (video sesi-12: "Eps 1072 | 4 CARA LEPAS DARI KEMISKINAN DI INDONESIA" — guru gembul)

> Sesi agent ini menjalankan alur §13 end-to-end LAGI setelah sandbox reset.
> Semua di bawah terverifikasi jalan di sesi ini. 12 klip full-coverage
> 0-1796.7s (29:57) selesai render + QA + upload dalam satu sesi.

## 22. Pemulihan sesi ini: model hilang dari upload/ofc-clip-kit/

- Sandbox reset LAGI — pola §16-17 tetap berlaku: `upload/` selamat.
- BARU: `upload/ofc-clip-kit/whisper.cpp/` TIDAK lagi berisi
  `ggml-small.bin` (487MB) — entah dibersihkan antar sesi. Solusi cepat
  terbukti: extract SATU FILE dari cache-pack root, cuma 7 detik:
  ```
  tar xzf upload/clip-kit-cache.tar.gz -C ofc-clip-kit/ whisper.cpp/ggml-small.bin
  ```
- Temuan kecepatan mount: rsync per-file LAMBAT (~1.1MB/s, 1.25GB = 3
  tool call × 10 menit), tapi BACA SEKUENSIAL FILE BESAR SANGAT CEPAT
  (dd test 3.3 GB/s). Jadi: kode via rsync, barang berat via tarball
  cache-pack — jangan rsync model/binary besar kalau bisa dari tarball.
- Verifikasi rsync: bandingkan `find . -type f -printf "%s %p\n" | sort`
  + jumlah byte source vs target (du bisa menyesatkan di ossfs).

## 23. Catatan operasional sesi ini

1. Kalibrasi whisper 1.02x realtime → part-len 460 → 4 bagian → merge
   4.095 kata mono 0 back-jump (video 29:57).
2. **Jingle intro di tengah video di-SKIP** (preseden baru): klip-02
   berakhir di "Yuk kita bahas." (289.91), klip-03 mulai SETELAH jingle
   musik 290.26-301.98 → start 302.00. Full-coverage tetap berlaku
   kecuali segmen non-bicara yang jelas-jelas jingle/musik.
3. Script QA baru masuk kit: `scripts/qa_clip.py` — QA piksel batch
   (ambil N frame + hitung piksel kuning #FFD60A area subtitle +
   auto-saran cek tetangga). 2 kasus LOW @ frame sampling keduanya
   micro-pause antar kata (§20.6 terkonfirmasi lagi) — cek tetangga
   + transkrip words-around sebelum diagnosa gagal.
4. Pola upload sesi ini: 21.6MB & 27.5MB → Contents API OK; sisanya
   (27-58MB) langsung `gh-push-big.sh` — semua sekali jalan cepat.
5. Urutan render topik terkuat dulu (§11): nyogok-PNS → pesta-nikah →
   rumus-4 → moge-vs-gerobak → 4-ciri → S1-baso → kafe → tanda-miskin
   → mobil-macet → naik-ojek → bahan-bakar → bandingkan-diri.
6. Zip kit di-refresh + di-upload di akhir sesi (55 file kode + docs +
   render-inputs; TETAP tanpa node_modules/whisper/out/raw.mp4/cache).

## 24. Checklist penutupan sesi (jangan ada yang tertinggal)

1. Semua klip ter-upload (12/12) ✓
2. `metadata.json` per sesi (format sesi-08/11, topik dirangkum detail) ✓
3. `README.md` repo: tambah baris tabel sesi ✓
4. `CARA-KERJA.md`: addendum sesi ✓ (file ini)
5. `ofc-clip-kit.zip` refresh + upload ✓
6. Scan token sebelum zip: `rg -l "github_pat_" scripts/ src/ *.md *.json` harus kosong ✓

---

# ADDENDUM SESI-14 (video sesi-13: "RESPECT GONTOR! Harusnya Ulama itu Kayak Gini!" — Felix Siauw)

> Sesi agent menjalankan alur §13 end-to-end setelah sandbox reset, TAPI dengan
> kondisi BARU: mount `upload/` KOSONG total (§16-17 tidak berlaku — tidak ada
> warisan sesi sama sekali, mulai dari zip kit + token dari chat). 6 klip
> full-coverage 0-918.65s (15:19) selesai render + QA + upload dalam satu sesi.

## 25. PENTING: bootstrap.sh PUNYA BUG cmake → bisa bikin cache-pack RUSAK

Kondisi awal sesi ini: zip kit fresh extract, `upload/` kosong. Run PERTAMA
bootstrap.sh gagal diam-diam di langkah cmake:

1. **Bug**: bootstrap menjalankan `pip install cmake` LANGSUNG — ditolak PEP 668
   (`externally-managed-environment`), padahal fix yang terbukti (§10.2) adalah
   venv + `python -m pip`. Script meng-ekspor PATH ~/.venv/bin tapi TIDAK bikin
   venv-nya kalau belum ada.
2. **Akibat berantai**: whisper.cpp ke-clone tapi gagal build (cmake absen) →
   bootstrap tetap lanjut → **cache-pack 145M dibuat dari state RUSAK** (isi
   whisper.cpp source tanpa binary/model). Run bootstrap berikutnya extract
   cache rusak itu → gagal dengan "Whisper folder exists but the executable
   (whisper.cpp/main) is missing. Delete whisper.cpp and try again."
3. **Fix yang terbukti jalan**:
   ```
   python3 -m venv ~/.venv
   ~/.venv/bin/python -m pip install cmake
   cd ofc-clip-kit && rm -rf whisper.cpp clip-kit-cache.tar.gz
   export PATH="$HOME/.venv/bin:$PATH"
   bash scripts/bootstrap.sh     # selesai ~143s, cache-pack 573M BENAR
   ```
4. **Cegah berulang**: kalau bootstrap kedua kalinya extract cache lalu mati
   di "executable is missing" → JANGAN diagnosis whisper-nya, itu cache-pack
   rusak (dibuat dari run gagal). Hapus whisper.cpp + cache-pack + pastikan
   cmake venv ada, ulang bootstrap. (Ide perbaikan bootstrap: jangan bungkus
   cache-pack kalau whisper binary belum ada — belum di-patch, masih tugas
   sesi berikutnya kalau mau.)

## 26. ffprobe packet-pts itu DECODE ORDER (B-frame) — WAJIB sort dulu

Cek CFR §3 dengan `packet=pts_time`: output ffprobe TIDAK urut waktu
(reordering B-frame). Analisis gap tanpa sort menghasilkan anomali palsu
(sesi ini: 14.916 "gap" palsu dari 22.023 paket). **Sort dulu pts-nya baru
hitung grid** — hasil benar sesi ini: 100.00% interval tepat di grid
1001/24000 (23.976fps CFR asli) → dipakai langsung tanpa re-encode.

## 27. Catatan operasional sesi ini

1. Video 918.65s (15:19) 720p CFR → 3 bagian transkrip (kalibrasi 0.90x,
   part-len 460) merge 2.127 kata mono 0 back-jump.
2. 6 klip full-coverage: 0-158.85 / 158.85-320.11 / 320.11-523.50 /
   523.50-718.37 / 718.37-851.53 / 851.53-918.65 — semua batas di ujung
   kalimat persis via words-around. Klip penutup 67.6s (pesan punch-line
   "ulama tak bisa disetir") — boleh di bawah 111s asalkan topik tuntas.
3. Urutan render prioritas topik terkuat: fatwa-jam-miliaran →
   ulama-tak-bisa-disetir → juru-bicara-penguasa → agama-dasar →
   pukulan-Konstantinopel → penggembala-dan-ayah.
4. QA piksel: 3 frame LOW di 2 klip — SEMUA ternyata micro-pause antar kata
   atau pop-in kata baru (kata mulai 0.07s sebelum frame sampling). Frame
   tetangga ±1-2s semua OK → §20.6 terkonfirmasi LAGI: selalu cek tetangga
   + transkrip sebelum diagnosa gagal.
5. Upload: 12.2MB & 26.5MB → Contents API OK; 30-45MB → git plumbing.
   lihat §28 untuk pola timeout baru.
6. Whisper model `ggml-small.bin` sekali build langsung ikut cache-pack
   573M (bootstrap normal) — simpan cache-pack ke `upload/` di akhir sesi
   (§21) supaya sesi berikutnya jalur A/B pemulihan tersedia lagi.

## 28. gh-push-big.sh: timeout tool call di tengah lazy-fetch — jangan panic

Run pertama `gh-push-big.sh` (44MB) **ke-timeout tepat saat lazy-fetch blob
repo** (tool call 5 menit). Dua artefak yang ditinggalkan:

1. `work/gh-push/.git/index.lock` — run berikutnya tolak jalan
   ("Another git process seems to be running").
2. 1.7GB blob yang SUDAH ke-fetch tetap ada di work dir.

**Fix terbukti**: `rm -f work/gh-push/.git/index.lock` → jalankan LAGI
script yang sama → push sukses CEPAT (blob yang sudah lokal tidak di-fetch
ulang). Jangan `rm -rf` work dir-nya — justru itu aset yang bikin retry cepat.

**Optimisasi sesi ini** (simpan di luar kit): `gh-push-session.sh` — duplikat
gh-push-big.sh yang TIDAK menghapus work-dir setelah push. Push pertama
tetap bayar lazy-fetch sekali (~4-6 menit), push berikutnya di sesi yang
sama hanya hitungan detik. Hapus work-dir manual di akhir sesi.
(Lokasi agent: /home/z/my-project/scripts/gh-push-session.sh — tidak ikut
zip kit, sesi berikutnya bikin ulang 1 menit.)

## 29. Layout pemulihan di upload/ yang ditinggalkan sesi ini

Sesi ini meninggalkan mount `upload/` dalam struktur BERBEDA dari §16-17
(optimasi §22: barang berat via tarball, kode via rsync):

- `upload/clip-kit-cache.tar.gz` (600MB) — whisper.cpp TER-BUILD + model
  small + node_modules + chrome headless (SAMA dengan cache-pack root kit).
- `upload/ofc-clip-kit/` — kode lengkap (src, scripts, public, render-inputs,
  config) + `out/clips` (6 klip final) + `out/transcripts` + `out/data` +
  `public/input/raw.mp4`. TANPA whisper.cpp/node_modules (sengaja dihapus —
  ambil dari tarball, jangan rsync per-file: ossfs rsync ~0.8MB/s TAPI cp
  sekuensial ~97MB/s — terukur sesi ini: 600MB = 6 detik).

**Alur pemulihan sesi berikutnya ( tercepat ):**
```
1. rsync -a upload/ofc-clip-kit/ ofc-clip-kit/           # kode+out, ~30s
2. tar xzf upload/clip-kit-cache.tar.gz -C ofc-clip-kit/ # whisper+model+nm+chrome, ~1-2 menit
3. tulis token baru ke work/.ghtoken
4. bash scripts/bootstrap.sh  # harus skip semua (verifikasi biner ada)
```
Jangan pakai Jalur A §17 apa adanya (rsync full) — itu menyalin
node_modules/whisper per-file LAMBAT; pakai kombo rsync-kode + tarball di atas.

---

# ADDENDUM SESI-15 (video sesi-14: "GJ 160 | ALASAN PRABOWO TIDAK BISA MELAWAN MAFIA HUKUM DI INDONESIA?" — guru gembul)

> Sesi agent ini menjalankan alur §13 end-to-end di workspace warisan (sandbox
> TIDAK di-reset: ofc-clip-kit/ + node_modules + whisper + chrome masih utuh —
> rsync §29 instan karena identik). 6 klip full-coverage 0-692.29s (11:32)
> selesai render + QA + upload dalam satu sesi. Temuan penting baru: §30.

## 30. PENTING: cache transcribe-parts TIDAK ter-key per video — WAJIB bersihkan saat ganti video

Sandbox warisan masih berisi `out/tmp/parts/part-XX.words.json` + `part-XX.wav`
dari video SESI LALU. `init` memang meng-extract `raw-full.16k.wav` baru, tapi
`part N` tetap bilang "sudah ada — skip" karena cek idempotensinya hanya
lihat ADA/TIDAK-nya file cache (tidak ada identitas video). Akibatnya `merge`
bisa menggabungkan TRANSKRIP VIDEO LAMA ke video baru — salah total tanpa
error.

**Aturan mulai sekarang**: saat memproses video BARU di workspace bekas
sesi lama (apapun jalur pemulihannya), SEBELUM `init` jalankan:
```
rm -f out/tmp/parts/part-*.wav out/tmp/parts/part-*.words.json \
      out/tmp/parts/calib-120s.wav out/transcripts/raw.json
```
(Ide perbaikan script: simpan hash/durasi video di parts-meta.json dan
invalidasi cache otomatis — belum di-patch, tugas sesi berikutnya.)

## 31. Catatan operasional sesi ini (semua terverifikasi jalan)

1. Video 692.29s (11:32) 720p **CFR 30fps asli** (full-scan 20.766 paket
   sorted-pts: 100% di grid 1/30s, 0 drift — §26 terkonfirmasi lagi) →
   langsung tanpa re-encode.
2. Download loader.to: kena `UND_ERR_SOCKET` di 92MB → retry sukses.
   Tool call bisa ke-timeout SETELAH file selesai ter-rename — selalu
   ffprobe dulu sebelum ulang download (§8 terkonfirmasi lagi).
3. Kalibrasi whisper 0.92x realtime → part-len 460 → 2 bagian → merge
   **1.495 kata** mono 0 back-jump.
4. 6 klip full-coverage 0-692.29s, semua batas di ujung kalimat persis via
   words-around: 0-128.14 / 128.14-231.06 / 231.06-376.00 / 376.00-472.72 /
   472.72-552.02 / 552.02-692.29 (user minta eksplisit: tidak boleh ada
   topik terpotong — video padat, batas diangka di ujung kalimat tuntas).
5. Urutan render prioritas topik terkuat dulu: clip-03 tantangan-prabowo
   (topik judul video) → clip-01 sidak-viral → clip-06 uang-haram →
   clip-02 → clip-05 → clip-04. Setiap klip: rm -rf out/segments →
   render-retry (masing-masing cukup 1-2 panggilan tool) → finalize →
   qa_clip.py → upload LANGSUNNG.
6. QA piksel: 3 frame LOW semuanya micro-pause antar kata / pop-in awal
   kata (§20.6 + §10.7 terkonfirmasi lagi) — frame tetangga ±1-2s semua
   OK, QA LULUS.
7. Upload: 19.5MB & 27.6-27.7MB → git plumbing session (§32); 33.4-39.5MB
   → git plumbing session. Semua sukses.
8. `qa_clip.py` kadang exit code -9/255 SETELAH print hasil lengkap
   (dibunuh saat cleanup) — output QA tetap valid, baca stdout-nya.

## 32. §28 dioptimalkan: gh-push-session.sh VERIFIKASI — push berikutnya 10 detik

`gh-push-big.sh` versi kit menghapus work dir setelah push → SETIAP push
bayar lazy-fetch blob repo (~8 menit di sesi ini, 2x). Varian §28 dibuat
di `/home/z/my-project/scripts/gh-push-session.sh` (TIDAK ikut zip kit —
sesi berikutnya bikin ulang 1 menit, atau copy dari addendum ini):

- WORK dir terpisah `work/gh-push-session/` yang TIDAK dihapus tiap push
  (+ auto `rm -f .git/index.lock` §28 sebelum jalan).
- Push pertama: 8m02s (lazy-fetch sekali). Push kedua dst: **10-11 detik**.
- Token PAT nyangkut di `work/gh-push-session/.git/config` (remote URL) —
  aman karena DI LUAR kit, tapi **WAJIB `rm -rf work/gh-push-session` di
  akhir sesi**.

---

# ADDENDUM SESI-16 (video sesi-15: "UST FELIX SIAUW TAJEM BANGET MULUTNYA! WENDI & ANDHIKA JADI NGERI!" — Ngobrol di WA Eps.73, Wendi Cagur)

> Sesi agent ini menjalankan alur §13 end-to-end setelah sandbox reset (root
> project kosong, mount `upload/` selamat seperti §16). Video TERPANJANG di
> project ini: 69:11 (4151.12s) → intro jingle suara air 0-457.5s di-skip
> (preseden §23.2), full-coverage ucapan 457.5-4151.12s = **24 klip** selesai
> render + QA + upload dalam satu sesi. User minta kit diambil DARI REPO.

## 33. Pemulihan versi "kit dari repo" — kombinasi tercepat terverifikasi

User eksplisit minta "ambil klip kit dari repo dan file cara kerja md".
Urutan pemulihan yang terbukti (~5 menit total):

1. `CARA-KERJA.md` + `ofc-clip-kit.zip` di-download dari root repo via
   Contents API → zip di-extract ke `ofc-clip-kit/` (zip TANPA folder
   induk — `mkdir -p ofc-clip-kit && unzip -d ofc-clip-kit/`).
2. whisper.cpp TER-BUILD + model: `cp -a upload/ofc-clip-kit/whisper.cpp
   ofc-clip-kit/` — 29 detik (model 487MB = 1 file besar sekuensial, cepat).
3. node_modules: **JANGAN `cp -a`** (file kecil ribuan → timed out 5 menit,
   terhenti setengah). Cache-pack tarball juga TIDAK berisi node_modules
   penuh — hanya `node_modules/.remotion/chrome-headless-shell`. Solusi
   tercepat: `npm ci` = **8,6 DETIK** (npm cache lokal masih hangat) lalu
   `tar xzf upload/clip-kit-cache.tar.gz -C . node_modules/.remotion`
   (7 detik) untuk chrome.
4. `bash scripts/bootstrap.sh` → harus skip semua (verifikasi).

## 34. ENOSPC terjadi 3x — daftar pemakan disk yang WAJIB dipantau

Disk sandbox cuma 9.9GB; render 24 klip (612MB sumber + ~1GB output)
menabrak batas 3x (gejala: render mati diam-diam errno -28 di
screenshotTask). Pembersihan yang terbukti, urut dampak:

1. **`/tmp/my-project` = 5.4GB SALINAN BASI** project dari sesi lalu (isi
   ofc-clip-kit 2.8G + upload 1.9G lama) — ditemukan sesi ini, HAPUS TOTAL
   (dir kosongnya "Operation not permitted", biarkan). SELALU cek
   `du -sh /tmp/*` saat ENOSPC.
2. `out/tmp/finalize/` — intermediat concat/audio per klip menumpuk ~1GB
   setelah ~20 finalize. Aman dihapus tiap beberapa klip (file final sudah
   di out/clips).
3. `out/tmp/parts/*.wav` + `work/raw-new.mp4` (duplikat raw.mp4) — ~850MB,
   aman dihapus setelah merge transkrip selesai.
4. Klip yang SUDAH ter-upload repo → hapus salinan lokalnya di out/clips
   (repo = source of truth; push terverifikasi OK dulu).
5. `/tmp/react-motion-render*` menumpuk tiap percobaan render gagal.

## 35. Catatan operasional sesi ini

1. Video 4151.12s 720p **CFR 25fps asli** (full-scan 103.776 paket sorted:
   100% grid 0.04s, 0 gap) → tanpa re-encode. loader.to smooth sekali jalan
   (612MB; tool call timeout SETELAH file selesai — ffprobe dulu, §8).
2. Kalibrasi whisper 0.66x realtime → part-len 460 → **10 bagian** → merge
   9.145 kata mono 0 back-jump. Bagian 0 hanya 86 kata (intro jingle ikut
   kepotong bagian pertama — normal).
3. 24 klip full-coverage ucapan, semua batas ujung kalimat via words-around.
   Klip 80-239s; clip-07 (94.8s) & clip-20 (80s) di bawah 111s boleh karena
   topik tuntas (preseden §27.2 klip 67.6s).
4. QA piksel: 9 klip ada frame LOW, SEMUA micro-pause antar kata/frasa
   cepat (§20.6 terkonfirmasi N-kali) — frame tetangga ±1s selalu OK.
   `qa_clip.py` butuh PATH file (bukan clip-id) + exit -9/255 setelah print
   hasil adalah normal (§31.8).
5. Upload: 19.5MB & 25-32MB juga lewat git plumbing (Contents API cuma
   dipakai metadata/README — file <5MB). `gh-push-session.sh` (§32) dibuat
   ulang di `scripts/` agent: push pertama lazy-fetch sekali, push 2-24
   masing-masing ~10 detik. 24 klip + metadata + README semua sukses.
6. Prioritas render topik terkuat dulu: logika-vs-perasaan → empati-rosul
   → pemimpin-bodoh → penutup → curhat-wendi → dst (urutan lengkap di
   metadata.json).

---

# ADDENDUM SESI-17 (video sesi-16: "JADI IMAM SHOLAT GAMAU, KENAPA JADI PRESIDEN REBUTAN?! | RADIO BAHLUL" — C8 Podcast)

> Sesi agent menjalankan alur §13 end-to-end setelah sandbox reset (kit diambil
> dari repo via zip + upload/ mount, pola §33). Dua pekerjaan besar dalam satu
> sesi: (1) UPGRADE KIT ke v1.1 (badge source, visual) per permintaan user,
> (2) 20 klip full-coverage 0-3325.05s selesai render + QA + upload. Temuan
> paling penting: §38 (render crash = disk) dan §39 (upload file besar).

## 36. Pemulihan & operasional transkrip sesi ini

1. Pemulihan §33 terkonfirmasi lagi: zip repo + `cp -a upload/ofc-clip-kit/
   whisper.cpp` + `npm ci` (8 dtk) + `tar xzf upload/clip-kit-cache.tar.gz
   node_modules/.remotion` + bootstrap skip-semua = total ~5 menit.
2. Video 3325.05s (55:25) 720p **CFR 50fps ASLI** (full-scan 166.249 paket:
   100.000% grid 1/50s) → tanpa re-encode. loader.to 855MB sekali jalan
   (ter-rename SEBELUM tool call mati — selalu ffprobe dulu, §8).
3. Kalibrasi whisper 0.90x → part-len 460 → 8 bagian. **Bagian 6 dua kali
   kebablas limit 10 menit** (load CPU sandbox naik-turun, kecepatan whisper
   per sesi TIDAK konstan — kalibrasi awal hanya patokan). Solusi baru yang
   terbukti: **split paruh** — potong wav bagian jadi 2 paruh (overlap 2s),
   whisper masing-masing paruh (~5 menit, muat di satu panggilan), gabung
   dengan aturan titik-tengah overlap (A: start < mid, B: start >= mid,
   waktu B digeser +mid-1). Skrip eksternal agent:
   `/home/z/my-project/scripts/part-split.mjs` (init|half|join). Hasil merge
   akhir tetap dicek seperti biasa (mono, 0 back-jump).
4. Merge 8 bagian = 8.665 kata mono 0 back-jump. §30 (bersihkan cache parts
   lama SEBELUM init) tetap WAJIB.
5. 20 klip full-coverage ucapan 0-2024 + 2110.95-3325.05 (iklan read Dilan
   Macak 2024-2110.95 di-SKIP, preseden §23.2); semua batas ujung kalimat
   via words-around; durasi 88-243 dtk.

## 37. UPGRADE KIT v1.1 (diminta user: "source:" + visual lebih bagus)

Perubahan yang masuk zip kit terbaru (semua tervalidasi TSC + QA piksel +
review VLM 8-9/10 sebelum dipakai produksi):

1. **Badge `source: <channel>`** pojok kiri-bawah — komponen baru
   `src/components/SourceBadge.tsx` (chip frosted + ikon play segitiga CSS
   murni, tanpa glyph font = bebas tofu). Nama channel mengalir:
   `render-inputs/clips.json` field `sourceChannel` → build-clips.mjs →
   `out/data/*.json` → komponen. Kosong = badge hidden otomatis.
2. **Washi tape** (`TapeCorners.tsx`) — 2 strip translucent menyilang tepi
   atas kartu, dirender DI LUAR kartu (wrapper terpisah dari div
   overflow-hidden) supaya boleh menjorok keluar tepi kartu.
3. **Watermark jadi chip pill** + titik aksen kuning; **progress bar pindah
   ke bawah** (pill + head dot kuning ala story IG, `position: 'bottom'`);
   **animasi intro** kartu (scale 0.962 + rise, spring — tanpa fade opacity
   supaya frame pertama tidak gelap); **subtitle pindah ke area kertas**
   (centerY 0.595 -> 0.71, sepenuhnya di bawah kartu — VPM merekomendasikan,
   tidak lagi menumpuk tepi kartu); VIDEO.centerY 0.44 -> 0.465;
   WATERMARK.top 84 -> 170 (ritme vertikal seimbang).
4. Semua nilai di `src/config.ts` (single source of truth); README kit
   di-update; versi package.json 1.1.0.
5. **QA visual upgrade wajib sebelum produksi**: render still via
   `qa-frames.mjs <id-test>` (buat data uji manual dari demo-words + video
   asli + sourceChannel), lalu cek piksel per elemen (script eksternal
   `scripts/qa_upgrade_check.py`) + 1-2 putaran review VLM
   (`z-ai vision -i frame.png`). Iterasi VLM: subtitle overlap -> geser
   centerY; spacing kurang seimbang -> fine-tune watermark/cardY.

## 38. PENTING: render crash "Target closed / Compositor SIGTERM" = DISK /tmp, bukan bug video

Gejala: satu klip (clip-13) gagal render 5+ percobaan, selalu mati di
frame tertentu (~35-55% segmen 0), error campuran "Protocol error
(Page.bringToFront): Target closed" + "Compositor quit with signal SIGTERM"
+ proxy 500 pada `?time=<t>`. Diagnosa keliru yang BUKAN penyebab:
- sumber video zona itu (ffmpeg extract frame OK, dan mini-klip 5 dtk
  yang MELINTASI zona itu render sempurna)
- memori (3.2GB bebas), timeout Remotion (sudah dinaikkan tanpa efek)

**Penyebab sebenarnya: disk /tmp mepet (~1.7GB bebas).** Render
concurrency 2 menaruh bundle (~830MB) + assets compositor (~800MB/proses)
di /tmp — makin sempit disk, makin cepat compositor mati diam-diam.
Fix yang terbukti: **bereskan disk sampai >=2.5GB** (hapus salinan lokal
klip yang SUDAH ter-upload §34.4, /tmp/remotion-*, out/tmp/finalize)
→ render langsung jalan normal sekali coba. Setelah itu: hapus klip lokal
otomatis begitu upload OK (sudah ditambahkan ke finish-one.sh agent).
Urutan diagnosa untuk sesi berikutnya: (1) df -h, (2) mini-klip uji di
zona crash, (3) baru curiga video/Remotion.

## 39. PENTING: upload file besar — git plumbing MATIKAN untuk repo sebesar ini; Contents API + CRF 26

Repo sudah ±550MB per sesi × 16 sesi. Yang terjadi di sesi ini saat coba
git plumbing (gh-push-session varian §32):

1. `git push` pada partial clone (blob:none) memicu **lazy-fetch SEMUA blob
   repo** (bukan sekadar delta-base): work dir bengkak 6.7-7.6GB → ENOSPC
   → fetch gagal diam-diam (stderr dibuang `2>/dev/null`, exit 128, LOG
   KOSONG — hati-hati mendiagnosa).
2. `GIT_NO_LAZY_FETCH=1` bikin `read-tree` instan (0.002s, 196KB) TAPI push
   tetap butuh verifikasi blob → "fatal: could not fetch <blob> from
   promisor remote". Jalan buntu dua arah — **jangan pakai git plumbing
   lagi untuk upload klip di repo ini**.
3. **Contents API (gh-upload.mjs) = satu-satunya jalur yang jalan**, batas
   empiris: ±40MB OK (39.5MB lolos), 63-92MB gagal 409 "Timed out
   validating the rule" (server-side, konsisten, bukan transient).
4. **Akar masalah ukuran: render Remotion default CRF 18 → klip ~3Mbps =
   0.4MB/s** (clip 100s = 40MB, clip 243s = 92MB). Fix permanen di kit:
   `renderMedia({crf: 26})` di render-segments.mjs (HATI-HATI: nama opsi
   `crf`, BUKAN `x264Crf` — salah nama = diam-diam diabaikan). Hasil: klip
   20-40MB langsung dari render, QA piksel kuning tetap LULUS.
5. Klip yang SUDAH kebesaran: re-encode `ffmpeg -crf 26 -preset medium`
   (243s = 39.3MB, terverifikasi). **`-preset veryfast` PATOLOGIS di CPU
   sandbox ini** (30 dtk video >5 menit encode, penyebab tidak ketemu;
   `medium` justru normal 0.3-0.43x realtime) — SELALU pakai medium.
6. Pipe `cmd | tail -N; echo $?` MENGEMBALIKAN exit code tail, bukan cmd —
   skrip upload/push harus log ke file (`> log 2>&1`) lalu `echo $?`.

## 40. Pola operasional rantai yang terbukti (20 klip dalam ~5 jam)

- Skrip agent `/home/z/my-project/scripts/`: `clip-cycle.sh` (usang, tergantikan),
  `finish-one.sh` (finalize → QA 3 titik + auto cek tetangga LOW → upload →
  bersih-bersih + HAPUS klip lokal setelah upload OK), `chain.sh <id-selesai|->
  <id-berikut>` (finalisasi klip A + langsung mulai render klip B dengan sisa
  waktu panggilan tool; marker `out/segments/.clip-id` mencegah reset segmen
  saat panggilan ulang — JANGAN rm -rf out/segments tanpa cek marker).
- Rata-rata: klip 100-243 dtk = 2-3 panggilan render + 1 panggilan chain.
  QA LOW di 5 klip semuanya micro-pause (§20.6 ke-N kalinya) — cek tetangga
  otomatis di finish-one.sh, semua LULUS tanpa intervensi.
- Prioritas render topik terkuat dulu (§11): imam-sholat-presiden (judul) →
  keadilan → circus-and-bread → pemimpin-bodoh-akhir-zaman → sirkel-F → dst.

# ADDENDUM SESI-18 (video sesi-17: "#ngopenk OBROLAN SERIUS YANG SEDIKIT ABSURD BERSAMA INDRA FRIMAWAN!" — Tirta PengPengPeng)

> Dua pekerjaan satu sesi lagi: (1) UPGRADE KIT v1.2 per permintaan user
> ("bakground lebih baik seperti kertas saja, ambil dari stock gratis online,
> dan reposition") + push zip pengganti, (2) 11 klip full-coverage 0-1137.4s
> selesai render + QA + upload.

## 41. UPGRADE KIT v1.2 — background kertas stock + reposition

1. **Pemulihan §33 masih valid** (~2 menit): zip repo → cp -a whisper.cpp
   (model 487MB ada di ROOT whisper.cpp, BUKAN models/ — SDK @remotion/
   install-whisper-cpp mencarinya di root; salin dua-duanya aman) → npm ci
   → tar chrome → bootstrap skip semua.
2. **Cari tekstur stock gratis** — urutan yang terbukti sesi ini:
   - z-ai image-search: upstream DOWN (400 Bad Request konsisten) — jangan
     buang waktu lama, langsung fallback
   - Wikimedia Commons: API jalan TAPI upload.wikimedia.org kena 429
     rate-limit IP (semua file 2KB HTML error) — CDN-nya diblok, bukan API
   - rawpixel.com langsung: 403 Cloudflare
   - **Openverse API JALAN**: `api.openverse.org/v1/images/?q=paper+texture
     &size=large&license_type=commercial` → JSON lengkap, preview
     rawpixel CC0 editor_1024 bisa di-curl langsung (6 kandidat sekaligus)
   - **YouTube oEmbed** (`youtube.com/oembed?url=...&format=json`) =
     cara tercepat dapat JUDUL + NAMA CHANNEL (author_name) tanpa yt-dlp —
     berguna utk field sourceChannel; jangan bergantung web-search (juga
     429/400 sesi ini)
3. **Preview 1024px cukup tajam** utk kanvas 1080x1920 dengan trik
   mirror-tile: Lanczos scale ke LEBAR 1080 (1.055x saja), lalu tile
   vertikal dgn flip-top-bottom bergantian (sambungan tak terlihat utk
   serat kertas stokastik). JANGAN object-fit: cover dari 1024x536
   (= stretch 3.5x). Skrip olah: scripts/make-paper-v2.py (agent);
   provenance+lisensi WAJIB dicatat di public/textures/paper-source.txt.
4. Pemilihan kandidat: contact sheet PIL + 1 putaran VLM ("pilih kandidat
   terbaik utk backdrop") — efektif & murah. Kandidat terpilih: kertas
   kraft oatmeal hangat (serat organik, bersih).
5. Reposition v1.2 (semua di config.ts): SOURCE badge pindah KIRI-bawah →
   **KANAN-bawah** (pojok kiri dipakai UI username/caption TikTok/Reels);
   VIDEO.centerY 0.465→0.455; SUBTITLE.centerY 0.71→0.70; WATERMARK.top
   170→156; PAPER.color #F1EBDD→#C9C2B7 (match tekstur baru) + vignette
   0.55→0.42 (tekstur asli sudah punya depth). SourceBadge.tsx: left→right.
6. QA visual upgrade (wajib sebelum produksi, pola §37.5): qa-make-test.mjs
   (BARU di kit) — slice 45s video asli + whisper + parsing persis
   transcribe.mjs (tokenLevelTimestamps + offsets ms + merge token) →
   out/data/qa-style-test.json; qa-frames render f0/15/60/240/900; cek
   piksel (tekstur semua zona, kuning kata aktif, scan chip kanan-bawah
   x580-1030 y1804-1858); VLM grid 3 frame → 8.5/10 LULUS.
   CATATAN: qa-frames.mjs pakai bundle lama kalau .remotion-bundle/ masih
   ada — SETELAH ubah style, HAPUS .remotion-bundle dulu supaya re-bundle.
7. Zip v1.2: JANGAN lupa public/fonts/ (Poppins) + CATATAN-REVISI.md —
   zip lama berisi keduanya; diff namelist zip lama vs baru sebelum push
   (scripts agent: diff namelist python zipfile). Scan token ketat
   `github_pat_[A-Za-z0-9_]{20,}` → push pengganti via gh-upload.mjs ✓.

## 42. loader.to sesi ini: koneksi mati diam-diam — bukan bug script

- Gejala: 3x proses download (node & curl) mati tanpa error di 90-151MB;
  pola ±75-100 detik per koneksi. Server savenow.to TIDAK dukung Range
  (GET Range diabaikan → stream penuh dari byte 0; resume mustahil).
- Solusi yang jalan: **buat sesi download BARU** (ajax/download.php lagi)
  → dapet server acak (leo3 mati di 151MB; **emma18** lolos full 159MB).
  Kalau mati lagi, ulangi start sampai kena server yang seumur hidup.
- TIP Diagnosis: `curl -D - -o /dev/null -H "Range: bytes=0-1023" <url>`
  dengan timeout — kalau hang/streaming penuh = server abaikan Range.
  Jangan tarik kesimpulan dari `alive=0` saja: **cek dulu ukuran file vs
  bitrate** — sesi ini curl "keluar" karena file SELESAI (159,109,120
  bytes = 1143.33s x 1.11Mbps persis), bukan mati. ffprobe .part dgn
  moov di depan langsung kasih durasi penuh.
- Video 1143.33s 720p **CFR 24fps asli** (27.439 paket 0 error) — tanpa
  re-encode. Whisper 0.94x → part-len 420 → 3 bagian (aman < 10 menit,
  tidak perlu split paruh §36.3). Merge 2.797 kata mono 0 back-jump.

## 43. Operasional 11 klip (rantai §40 dipakai ulang)

- 11 klip full-coverage ucapan 0-1137.4s, semua batas ujung kalimat via
  words-around; durasi 55-172.5 dtk. Prioritas render: vo2max (judul
  kuat) → dokter-memangsa → breakup-gym → dst (urutan lengkap di
  metadata.json).
- Skrip agent chain dibuat ulang (sandbox fresh): scripts/finish-one.sh
  (finalize → QA 3 titik dgn titik proporsional durasi + auto tetangga
  LOW → upload → bersih + HAPUS klip lokal) & scripts/chain.sh (finish A
  + render B budget sisa waktu; render log → file, tail saja ke stdout).
- Pola panggilan stabil: klip ≤96 dtk selesai 1 panggilan render; klip
  128-172 dtk butuh 2 panggilan (marker .clip-id resume pending). Exit
  -9 di akhir panggilan setelah "semua pending selesai" = timeout tool
  biasa, BUKAN gagal — cek dulu baris terakhir log sebelum ulang.
- QA: hanya clip-06 ada 1 titik LOW, tetangga ±1s semua OK (micro-pause
  §20.6 ke-N kalinya) — upload tetap jalan. clip-08 QA 3/3 langsung OK.
- Upload semua via Contents API (7.4-23.0MB @ CRF 26 — batas §39 aman).
  11 klip + metadata.json + README row terverifikasi di repo.

## 44. Checklist penutupan sesi (semua ✓ sesi ini)

zip kit v1.2 ter-push pengganti ✓ · 11 klip + metadata + README ✓ ·
lokal klip dihapus pasca-upload ✓ · out/tmp/finalize dibersihkan ✓ ·
transcript raw.json dipertahankan utk audit ✓ · PAT tetap hanya di
work/.ghtoken (scan ketat sebelum tiap push) ✓.
