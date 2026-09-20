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

## Spesifikasi Output

- Resolusi: 1080x1920 (9:16), 30 fps
- Caption: popup sederhana, 2-5 kata per grup, highlight kuning kata aktif (karaoke)
- Watermark: **ofkiehme** (atas-tengah)
- Durasi per clip: 30 detik - 3 menit (1 topik = 1 clip)
