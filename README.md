# TPT-01 — Edge ANPR & KTP Integration with CSMS

Repository ini berisi program edge untuk mendeteksi plat nomor kendaraan, membaca Nomor Induk Kependudukan (NIK) dari KTP, serta mengirimkan event hasil deteksi plat ke server CSMS.

Proyek ini dikembangkan untuk menjalankan pemrosesan di sisi perangkat lokal (*edge processing*) dan menghubungkan hasil deteksi dengan sistem pemantauan serta autentikasi akses parkir.

## 1. Project Overview

Proyek terdiri dari tiga komponen utama:

| Komponen | Fungsi | Status |
|---|---|---|
| `edge_anpr/` | Deteksi dan pembacaan plat nomor kendaraan melalui kamera | Berhasil diuji secara lokal |
| `edge_ktp/` | OCR untuk membaca NIK dari KTP dan menghasilkan event JSON | Berhasil diuji secara lokal |
| `proto/` | Pengiriman event ANPR ke API CSMS | Digunakan untuk integrasi ANPR |

**Catatan status:** Integrasi pengiriman event KTP ke server CSMS belum selesai. Program KTP saat ini menghasilkan event lokal dalam format JSONL. Server harus memiliki endpoint dan kontrak data yang sesuai sebelum integrasi KTP dapat dilakukan.

## 2. Repository Structure

```text
TPT-01/
├── edge_anpr/
│   ├── edge_main.py
│   ├── capture.py
│   ├── plate_utils.py
│   ├── aggregator.py
│   ├── requirements.txt
│   └── output/
│
├── edge_ktp/
│   ├── ktp_main.py
│   ├── capture.py
│   ├── ktp_reader.py
│   ├── nik_utils.py
│   ├── nik_aggregator.py
│   ├── requirements.txt
│   ├── README.md
│   ├── EVENT_SCHEMA.md
│   └── output/
│
├── proto/
│   └── sender_csms.py
│
└── README.md
```

Struktur di atas menggambarkan komponen utama yang digunakan. File tambahan, dokumentasi, dan folder pengujian dapat tersedia sesuai versi repository.

## 3. Requirements

- Windows dengan Command Prompt (CMD).
- Python yang dapat dijalankan melalui perintah `python`.
- Webcam atau kamera yang kompatibel.
- Koneksi internet untuk komunikasi dengan server CSMS.
- Server CSMS dan tunnel ngrok yang aktif untuk pengujian integrasi.
- Dependensi Python sesuai `requirements.txt` masing-masing komponen.

Periksa instalasi Python:

```cmd
python --version
```

## 4. Running Edge ANPR

ANPR digunakan untuk membaca plat nomor kendaraan melalui kamera.

### Install dependencies

Buka CMD, kemudian jalankan:

```cmd
cd /d C:\Users\Windows10\proyek-anpr\edge_anpr
python -m pip install -r requirements.txt
```

### Start camera and detection

```cmd
python edge_main.py --source 0 --device-id gerbang-1
```

Keterangan:
- `--source 0`: menggunakan kamera default.
- `--device-id gerbang-1`: identitas perangkat edge.
- Program menampilkan preview kamera dan menjalankan proses deteksi.

Hasil deteksi ditulis ke:

```text
edge_anpr/output/events.jsonl
```

Program ini juga mendukung sumber video atau RTSP sesuai implementasi yang tersedia.

## 5. Running Edge KTP

Edge KTP digunakan untuk membaca NIK dari KTP menggunakan OCR.

### Install dependencies

```cmd
cd /d C:\Users\Windows10\proyek-anpr\edge_ktp
python -m pip install -r requirements.txt
```

### Start camera and OCR

```cmd
python ktp_main.py --source 0 --width 1280 --height 720 --show
```

Keterangan:
- `--source 0`: menggunakan kamera default.
- `--width 1280 --height 720`: mengatur resolusi yang diminta.
- `--show`: menampilkan preview kamera.

### Recommended testing procedure

1. Pastikan tulisan NIK terlihat jelas.
2. Pegang kartu dengan stabil selama sekitar 2–3 detik.
3. Hindari pantulan cahaya dari hologram atau laminasi.
4. Turunkan kartu dan tunggu proses penyelesaian event.

Program menggabungkan hasil pembacaan dari beberapa frame untuk mengurangi kesalahan OCR dan mencegah event berulang untuk kartu yang sama selama periode cooldown.

Hasil event disimpan ke:

```text
edge_ktp/output/events.jsonl
```

Format event dijelaskan lebih lanjut di `edge_ktp/EVENT_SCHEMA.md`.

**Penting:** Program hanya memeriksa struktur NIK dan kualitas pembacaan OCR. Program tidak membuktikan keaslian KTP atau validitas NIK di database resmi.

## 6. Sending ANPR Events to CSMS

Komponen `proto/sender_csms.py` digunakan untuk membaca event ANPR dan mengirimkannya ke API CSMS.

### Configure API key

Sender membaca API key dari environment variable `API_KEY`. Jika tidak tersedia, script menggunakan nilai default untuk pengembangan.

Sebelum menjalankan sender, atur API key yang diberikan oleh pengelola server di CMD:

```cmd
set API_KEY=ISI_API_KEY_DARI_PENGELOLA_SERVER
```

Ganti placeholder tersebut dengan key yang benar. Jangan commit key ke GitHub atau membagikannya melalui screenshot.

### Start sender

Buka CMD baru:

```cmd
cd /d C:\Users\Windows10\proyek-anpr\proto
```

Jalankan sender dengan file event ANPR:

```cmd
python sender_csms.py --file "C:\Users\Windows10\proyek-anpr\edge_anpr\output\events.jsonl" --slot SLOT_A1 --url "https://YOUR-NGROK-HOST/api/v1/auth/anpr" --interval 1
```

Ganti `YOUR-NGROK-HOST` dengan hostname ngrok aktif milik pengelola server.

Parameter penting:
- `--file`: lokasi file event ANPR.
- `--slot`: slot parkir yang dikenali server, misalnya `SLOT_A1` atau `SLOT_A2`.
- `--url`: endpoint API autentikasi ANPR.
- `--interval 1`: interval pemeriksaan file dalam detik.
- `--max-age`: batas usia event dalam detik; default 30 detik.
- `--ev-hint`: mode pengiriman informasi klasifikasi EV.

Sender memproses event baru dan mencatat posisi baca pada file `.sent`. Event yang sudah terlalu lama akan dilewati untuk mencegah pengiriman event lama yang dapat memicu keputusan akses yang tidak sesuai.

Untuk melihat seluruh parameter yang tersedia:

```cmd
python sender_csms.py --help
```

## 7. CSMS Dashboard

Dashboard yang digunakan saat pengujian:

```text
https://YOUR-NGROK-HOST/monitor
```

Endpoint API ANPR:

```text
https://YOUR-NGROK-HOST/api/v1/auth/anpr
```

Ganti `YOUR-NGROK-HOST` dengan hostname tunnel yang aktif. Hostname ngrok dapat berubah ketika tunnel dibuat ulang.

Server dan tunnel harus aktif sebelum pengujian pengiriman dilakukan.

## 8. Current Integration Status

### Completed

- Program ANPR dapat membuka kamera dan menjalankan deteksi plat.
- Program KTP dapat membuka kamera dan membaca NIK melalui OCR.
- Kedua program menghasilkan event lokal dalam format JSONL.
- Sender ANPR tersedia untuk mengirim event ke endpoint CSMS.
- Dashboard CSMS berhasil dibuka dan data plat terbaru dilaporkan tampil saat pengujian.

### Remaining work

- Mengintegrasikan event KTP dengan server CSMS.
- Memastikan endpoint KTP tersedia di sisi server.
- Menyepakati format request, autentikasi, dan respons untuk event KTP.
- Menguji pembaruan data end-to-end dan menangani error pengiriman.
- Menentukan kebijakan retensi dan penghapusan data NIK di sisi edge dan server.

## 9. Troubleshooting

### Camera does not open

Pastikan kamera tidak digunakan aplikasi lain dan periksa output CMD. Tes kamera terpisah dapat digunakan untuk memastikan perangkat kamera dapat dibuka.

### Missing Python module

Jalankan instalasi dependensi dari folder komponen yang sesuai:

```cmd
python -m pip install -r requirements.txt
```

### ANPR event is skipped as stale

Sender ANPR melewati event yang lebih tua daripada batas `--max-age`. Uji dengan event baru dan pastikan waktu perangkat benar.

### API authentication error

Periksa `API_KEY`, endpoint, dan konfigurasi autentikasi server. Jangan membagikan key dalam log atau repository.

### KTP event does not reach CSMS

Sender ANPR saat ini mengharapkan field plat seperti `plate` dan tidak menerima event KTP yang berisi field `nik`. Diperlukan integrasi terpisah atau perubahan yang disepakati pada sisi server.

## 10. Data Privacy and Security

Event KTP dapat berisi NIK lengkap, yang merupakan data pribadi. Terapkan langkah berikut:

- Jangan commit `events.jsonl`, foto KTP, atau hasil OCR berisi NIK ke GitHub.
- Jangan commit API key, token, password, atau file `.env`.
- Gunakan HTTPS untuk pengiriman data.
- Batasi akses terhadap file event dan log.
- Tentukan masa penyimpanan dan mekanisme penghapusan data.
- Gunakan data sintetis atau data uji yang tidak mengidentifikasi individu selama pengembangan.
- Gunakan data KTP asli hanya jika ada dasar pemrosesan yang sah, tujuan yang jelas, dan pengamanan yang sesuai.

## 11. Handover Notes

Untuk melanjutkan proyek:

1. Baca README ini terlebih dahulu.
2. Jalankan dan verifikasi ANPR secara lokal.
3. Jalankan dan verifikasi KTP secara lokal.
4. Pastikan sender ANPR membaca file event dari `edge_anpr`, bukan dari `edge_ktp`.
5. Pastikan server CSMS dan tunnel aktif sebelum menguji pengiriman.
6. Jangan menganggap integrasi KTP selesai sebelum endpoint dan pengiriman end-to-end berhasil diuji.

Dokumen handover tambahan dapat digunakan sebagai panduan menjalankan CMD dan melakukan troubleshooting.

---

**Project:** TPT-01  
**Components:** Edge ANPR, Edge KTP OCR, CSMS integration  
**Status:** Local detection operational; ANPR integration tested; KTP server integration pending.
