# clip-sessions

Penyimpanan sementara (temporary storage) untuk semua output clip editing vertikal **9:16 (1080x1920)** dari project editing **ofkiehme**.

## Struktur

Setiap sesi editing = 1 folder, format penamaan: `sesi-<nomor>-<nama-video>/`

Di dalam tiap folder sesi:
- `clip-XX-<judul-topik>.mp4` — clip jadi (H.264 + AAC)
- `metadata.json` — info sumber video, tanggal, daftar clip & durasi

## Daftar Sesi

| Sesi | Sumber | Jumlah Clip |
|------|--------|-------------|
| [sesi-01-prof-zulys-yntv](./sesi-01-prof-zulys-yntv) | Podcast Prof. Zulys (YNTV) — youtu.be/z07M-QFMV04 | 7 clip |
| [sesi-02-podcast-lari-polusi-jakarta](./sesi-02-podcast-lari-polusi-jakarta) | Podcast lari saat polusi Jakarta — untitled.mp4 | 1 clip (utuh 2:38) |
| [sesi-03-podcast-kesehatan-pencernaan](./sesi-03-podcast-kesehatan-pencernaan) | Podcast kesehatan & pencernaan — untitled2.mp4 | 1 clip (utuh 3:37) |
| [sesi-04-podcast-serba-serbi-dokter-forensik](./sesi-04-podcast-serba-serbi-dokter-forensik) | Podcast Serba Serbi Dokter Forensik — Raditya Dika — youtu.be/g3Zd0F7CSuc | 10 clip (utuh 56:00, 1 topik tuntas = 1 clip) |

## Spesifikasi Output

- Resolusi: 1080x1920 (9:16), 30 fps
- Caption: popup sederhana, 2-5 kata per grup, highlight kuning kata aktif (karaoke)
- Watermark: **ofkiehme** (atas-tengah)
- Durasi per clip: mengikuti durasi topik — 1 topik tuntas = 1 clip (tidak dipotong di tengah topik)
