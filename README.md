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
| [sesi-05-karnaval-pekalongan-felix-siauw](./sesi-05-karnaval-pekalongan-felix-siauw) | Bahas Karnaval Pekalongan "Satanic" — Felix Siauw — youtu.be/Jd5hpeoSSqA | 8 clip (utuh 24:10, 1 topik tuntas = 1 clip) |
| [sesi-06-king-ronald-ancaman-penipu](./sesi-06-king-ronald-ancaman-penipu) | KING RONALD TERANCAM MASUK PENJARA — youtu.be/rviv8OIFg98 | 8 clip (utuh 20:32) |
| [sesi-07-nessie-judge-konspirasi-bencana](./sesi-07-nessie-judge-konspirasi-bencana) | NESSIE JUDGE GA PAKAI SENSOR — Bencana Bisa Dimanipulasi?? — youtu.be/16ruF24xR1s | 1 clip (utuh 52:59) |
| [sesi-08-dunning-kruger-felix-siauw](./sesi-08-dunning-kruger-felix-siauw) | Dunning-Kruger Effect Harus Direvisi! — Felix Siauw — youtu.be/AL-b8wznH0s | 8 clip (utuh 12:52) |
| [sesi-09-bobon-bantu-orang-politik](./sesi-09-bobon-bantu-orang-politik) | Bobon Mau Bantu Lebih Banyak Orang Lewat Politik — youtu.be/rnqvXYrVw4w | 9 clip (utuh 51:29) |
| [sesi-10-menolak-pakar-npd](./sesi-10-menolak-pakar-npd) | Menolak Pakar Termasuk NPD Nggak Sih? — Felix Siauw — youtu.be/3SkVPuJnGBI | 5 clip (utuh 13:02) |
| [sesi-11-ray-restu-ngomongin-wapres](./sesi-11-ray-restu-ngomongin-wapres) | Kenapa orang-orang pada ngomongin Mas Wapres? — Ray Restu Fauzi — youtu.be/Z4DgIN7JkEY | 11 clip (utuh 31:44) |

## Spesifikasi Output

- Resolusi: 1080x1920 (9:16), 30 fps
- Caption: popup sederhana, 2-5 kata per grup, highlight kuning kata aktif (karaoke)
- Watermark: **ofkiehme** (atas-tengah)
- Durasi per clip: mengikuti durasi topik — 1 topik tuntas = 1 clip (tidak dipotong di tengah topik)
