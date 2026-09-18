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

## Spesifikasi Output

- Resolusi: 1080x1920 (9:16), 30 fps
- Caption: popup sederhana, 2-5 kata per grup, highlight kuning kata aktif (karaoke)
- Watermark: **ofkiehme** (atas-tengah)
- Durasi per clip: 30 detik - 3 menit (1 topik = 1 clip)
