# Gatekeeper System for PT. Internusa Food

## 1. Gambaran Umum
Gatekeeper adalah sistem verifikasi dan pelacakan digital modular untuk PT. Internusa Food, ditempatkan di loading dock sebagai "penjaga gerbang" untuk memastikan palet dimuat ke truk sesuai Sales Order (SO) dari Accurate. Tujuannya menghilangkan ketidaksinkronan data antara Accurate (agregat) dan WMS (real-time), menyediakan bukti digital (foto, log, timestamp, video CCTV), dan menghasilkan laporan pratinjau untuk import ke Accurate sebagai Delivery Order (DO).

- **Masalah yang Diatasi**:
  - Ketidaksesuaian jumlah barang dengan SO (e.g., palet 48 kardus, SO butuh 40).
  - Kurangnya bukti visual/digital untuk komplain pembeli.
  - Proses manual rawan error dan lambat.
- **Fokus Utama**: Lacak palet berdasarkan SO, karena detail barang (jumlah kardus, jenis, batch, expiry) ditangani produksi/WMS. Mulai dengan uji coba Excel/CSV, skalabel ke API/RFID.
- **Manfaat**:
  - Kurangi komplain pembeli hingga 80% dengan bukti digital.
  - Tingkatkan efisiensi gudang via otomatisasi tracing.
  - Skalabel untuk integrasi API Accurate/WMS.
- **Asumsi Dummy untuk RFID Tag/Barcode** (dari produksi, disimpan di WMS):
  - Jenis barang (e.g., "Makanan XYZ", "Minuman ABC").
  - Waktu produksi (e.g., "2025-09-01 10:00").
  - Kadaluarsa (e.g., "2026-03-01").
  - Jumlah kardus (e.g., 48/palet, satu jenis barang).
  - Tambahan: Batch number ("BATCH-001"), supplier ID ("SUP-XYZ").

## 2. Arsitektur Sistem
Arsitektur client-server memisahkan frontend (UI) dan backend (logika bisnis).

- **Frontend (Aplikasi Web)**:
  - Dibangun dengan React + TypeScript, berjalan di browser (PC/laptop/tablet di dock).
  - Tangani UI: dashboard (live video, notifications), form Load Session, tabel palet dengan RFID scan.
  - Komunikasi via REST API ke backend.
- **Backend (Server Aplikasi)**:
  - Python dengan FastAPI (cepat, async, auto-docs).
  - Kelola logika bisnis, otentikasi (role: operator vs admin), database, CCTV (rekam/stream), verifikasi SO vs WMS, dan API.
  - Database: PostgreSQL (produksi), SQLite (dev).
- **Integrasi Eksternal**:
  - WMS: Ambil data palet via CSV/API.
  - Accurate: Ekspor CSV untuk DO.
  - CCTV: Rekam via IP camera (RTSP) dengan OpenCV.
  - RFID: Prioritas UHF RFID (EL-UHF-RC4-2 TCP/IP, jarak 4-5m, EPC Gen2) untuk scan otomatis palet; barcode scanner (USB/Bluetooth) sebagai fallback.

**Diagram Alur Sederhana**:
```
Produksi → WMS Rack (FIFO, catat RFID tag/barcode) → Loading Dock (Gatekeeper: Verifikasi SO, Scan RFID/Barcode, Catat Jumlah, Bukti Digital) → Accurate (Ekspor DO) → Pengiriman ke Pembeli
```

## 3. Alur Kerja Sistem
Proses dimulai saat tim warehouse terima SO dari Accurate dan siapkan palet dari WMS di dock. Gatekeeper verifikasi via CSV/Excel import untuk SO/WMS.

**Tahap 1: Persiapan Sesi Pemuatan (Loading Session)**  
1. Operator buka Gatekeeper di browser.  
2. Buat "Sesi Pemuatan Baru" dengan input:  
   - Nomor Plat Mobil: Otomatis via OCR (OpenCV) jika kamera ada; fallback manual.  
   - Nama Pengemudi: Manual.  
   - Foto Pengemudi: Via webcam/smartphone di frontend.  
   - Nomor SO: Manual atau import CSV/Excel (dari Accurate, termasuk tipe/jumlah barang).  
3. Sistem catat waktu mulai sesi, trigger CCTV jika aktif.

**Tahap 2: Proses Pemuatan dan Pemindaian Palet**  
1. Untuk setiap palet dari WMS, operator scan ID Palet via UHF RFID reader (EL-UHF-RC4-2, otomatis, TCP/IP) atau barcode scanner (manual fallback).  
2. ID Palet auto-tambah ke PalletsTable di frontend, ambil data WMS (e.g., 48 kardus XYZ).  
3. Penanganan Perbedaan Jumlah:  
   - Sistem tampilkan info palet (e.g., 48 kardus).  
   - Bandingkan dengan SO (e.g., butuh 40 kardus).  
   - Operator input jumlah aktual dimuat (e.g., 40) via tabel.  
   - Sisa (e.g., 8 kardus) ditangani manual di luar Gatekeeper (pindah ke palet khusus, catat ke WMS).  
4. Untuk truk besar (40+ palet), RFID reader scan bulk dari pintu belakang post-loading untuk verifikasi (e.g., 20 palet ke Truk A, 20 ke B).

**Tahap 3: Penyelesaian Sesi**  
1. Setelah semua palet sesuai SO dimuat, operator lampirkan bukti:  
   - Unggah foto muatan (frontend file upload).  
   - Catat tautan CCTV (backend generate MP4 dengan overlay SO/Palet_ID).  
2. Tandatangan Digital:  
   - Operator (PT. Internusa Food) dan Driver (penerima) tanda tangan via canvas (react-signature-canvas), simpan sebagai image.  
   - RFID verifikasi bulk (opsional) untuk konfirmasi palet di truk.  
3. Operator tekan "Selesaikan Sesi", status "Done", CCTV stop.

**Tahap 4: Pelaporan dan Verifikasi**  
1. Admin terima notifikasi (email/SMS/in-app) sesi selesai.  
2. Review di dashboard: Daftar palet, jumlah kardus, foto, signature, video CCTV.  
3. Ekspor:  
   - Excel/CSV: Untuk verifikasi sebelum import DO ke Accurate.  
   - PDF: Bukti pengiriman dengan detail, foto, signature, QR code ke video (kirim ke pembeli).

## 4. Tumpukan Teknologi (Tech Stack)
**Frontend**:  
- Framework: React + Vite  
- Bahasa: TypeScript  
- Styling: TailwindCSS  
- API: React Query (@tanstack/react-query)  
- Routing: react-router-dom  
- Chart: Chart.js + react-chartjs-2  
- Ikon: lucide-react  
- Opsional: hls.js (video HLS), react-use-websocket (real-time RFID)  

**Backend**:  
- Framework: FastAPI (async, cepat)  
- Database: PostgreSQL (produksi), SQLite (dev)  
- CCTV: OpenCV (rekam/overlay), Mediamtx (stream multiple dock)  
- Laporan: Pandas + openpyxl (Excel), FPDF (PDF)  
- Notifikasi: smtplib (email), Twilio (SMS)  
- RFID: EL-UHF-RC4-2 (TCP/IP, jarak 4-5m, EPC Gen2), pyserial/socket untuk komunikasi  
- Lainnya: Axios (API calls)

## 5. Requirements
- **Hardware**: PC/laptop di dock, IP camera (RTSP), UHF RFID reader (EL-UHF-RC4-2 TCP/IP untuk prototyping, jarak 4-5m; opsional EL-UHF-RF014 untuk truk besar, jarak 15m, 4 antenna ports), tags pasif (anti-metal), barcode scanner (fallback).  
- **Software**: Node.js 18+ (frontend), Python 3.10+ (backend).  
- **Lingkungan**: Lokal (dev), cloud (AWS/Heroku, produksi).

## 6. Installation
1. Clone: `git clone https://github.com/mrdon/gatekeeper-internusa.git`  
2. Frontend: `cd frontend; npm i; npm run dev` (proxy via VITE_API_BASE_URL=/api)  
3. Backend: `cd backend; pip install fastapi uvicorn pandas openpyxl fpdf opencv-python pyserial; uvicorn main:app --reload`

## 7. Usage
- Operator: Buat/load session via /loads route, scan RFID/barcode, input jumlah kardus.  
- Admin: Review/export via dashboard, verifikasi sebelum import DO.

## 8. Development Notes
- **Prototyping**: Gunakan EL-UHF-RC4-2 (TCP/IP) untuk uji coba 1 dock (SO kecil, 10-20 palet). Upgrade ke EL-UHF-RF014 untuk truk besar (40+ palet, jarak baca 15m).  
- **Security**: AES enkripsi untuk data sensitif, audit trail (timestamp per aksi).  
- **Testing**: Uji 1-2 minggu dengan data dummy, fokus verifikasi bulk palet di truk.  
- **Ekspansi**: Integrasi ML (OCR plat mobil, deteksi anomali video), API Accurate/WMS.  
- **Contributors**: Mr. Don (pemilik), Bejo (developer, ekspert Python/C++).

## 9. License
MIT License.

## 10. Contact
Hubungi Bejo via [email protected] untuk feedback.
