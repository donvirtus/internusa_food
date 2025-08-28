# Gatekeeper System for PT. Internusa Food

## Overview
Gatekeeper adalah sistem modular berbasis software yang dirancang untuk PT. Internusa Food guna mengatasi masalah ketidaksinkronan data pasca-barang keluar gudang menuju pengiriman ke buyer. Sistem ini berfungsi sebagai "penjaga gerbang" di loading dock, memverifikasi dan mencatat detail muatan ke kendaraan pengangkut, menghasilkan dokumen untuk review sebelum import ke Accurate, dan menyediakan bukti digital (termasuk rekaman CCTV) untuk menangani komplain buyer.

**Tujuan Utama:**
- Tracking akurat: Mengetahui barang apa yang dimuat ke truk mana.
- Preview data: Menghasilkan laporan untuk review manual sebelum import ke Accurate (surat jalan).
- Bukti anti-komplain: Memberikan dokumentasi digital (foto, log, timestamp, dan video CCTV) untuk buyer.
- Pendekatan bertahap: Uji coba via Excel tanpa ganggu sistem existing (Accurate/WMS).

**Masalah yang Diatasi:**
- Ketidaksinkronan stok antara Accurate (agregat) dan WMS (real-time).
- Kurangnya bukti visual/digital, menyebabkan komplain buyer soal barang kurang/tidak sesuai.
- Proses manual yang rentan error.

**Manfaat:**
- Kurangi komplain buyer hingga 80% dengan bukti digital dan video.
- Tingkatkan efisiensi operasional gudang.
- Skalabel untuk integrasi API atau RFID di masa depan.

## Features
### Fitur Utama
- **Tracking Muatan**:
  - Catat: Nomor mobil pengangkut (contoh: B 1234 XYZ), nama driver, nomor stock opname (SO-2025-001), detail barang (nama, jumlah, batch, expiry).
  - Tambahan: Timestamp loading, foto muatan, signature digital operator, dan rekaman CCTV selama proses loading.
- **Generasi Dokumen**:
  - Excel/CSV file untuk preview data muatan sebelum import ke Accurate.
  - PDF bukti muatan dengan foto, log detail, dan link ke rekaman CCTV.
- **Notifikasi Otomatis**:
  - Email/SMS ke admin internal untuk review data (dengan Excel, PDF, dan link video).
  - Email/SMS ke buyer dengan bukti digital setelah approval.
- **Integrasi CCTV**:
  - Rekam proses loading via IP camera (RTSP stream) di loading dock.
  - Auto-overlay timestamp dan stock opname pada video.
  - Simpan klip MP4, link ke output preview untuk review.
- **Integrasi RFID (Opsional)**:
  - Scan otomatis tag RFID pada barang/palet untuk tracking real-time.
- **Dashboard Monitoring (Opsional)**:
  - Web-based view untuk status muatan dan playback video CCTV.

### Fitur Keamanan
- Enkripsi AES untuk log, bukti digital, dan metadata video.
- Role-based access: Operator hanya input, admin approve/export.
- Audit trail dengan timestamp untuk setiap aksi, termasuk akses video.

## Requirements
- **Hardware**:
  - Laptop/PC untuk operator gudang.
  - Ponsel dengan kamera untuk foto muatan.
  - IP camera (contoh: Hikvision) dengan RTSP support untuk rekaman CCTV.
  - Opsional: RFID reader (handheld/fixed) dan tag RFID pasif.
- **Software**:
  - Python 3.10+.
  - Library: `pandas`, `openpyxl`, `fpdf`, `smtplib`, `twilio` (SMS), `pyserial` (RFID), `opencv-python` (CCTV).
  - Opsional: Flask/Django untuk dashboard, Firebase untuk push notification.
- **Lingkungan**: Lokal (laptop/Raspberry Pi) atau cloud (AWS/Google Cloud) untuk storage video.

## Installation
1. **Clone Repository**:
   ```bash
   git clone https://github.com/mrdon/gatekeeper-internusa.git
   cd gatekeeper-internusa
   ```

2. **Install Dependencies**:
   ```bash
   pip install pandas openpyxl fpdf twilio pyserial flask opencv-python
   ```

3. **Konfigurasi**:
   - Buat file `config.py` dengan:
     - SMTP server (email).
     - Twilio credentials (SMS).
     - Port RFID reader (jika digunakan).
     - RTSP URL untuk CCTV (contoh: `rtsp://username:password@camera_ip:554/stream`).
   - Buat folder `uploads/` untuk simpan foto dan video.

4. **Jalankan Script**:
   ```bash
   python main.py
   ```

## Usage
### Alur Kerja Uji Coba (Manual via Excel dengan Preview)
1. **Input Data**:
   - Operator masukkan data via form Excel atau app sederhana:
     - Nomor mobil, nama driver, nomor stock opname, detail barang.
     - Timestamp (otomatis), foto muatan (upload), signature digital.
     - Rekaman CCTV otomatis di-trigger saat proses loading dimulai.
2. **Proses Data**:
   - Script generate `outbound_data.xlsx` untuk preview (termasuk link ke video CCTV).
   - Generate `bukti_muatan.pdf` dengan detail muatan, foto, dan QR code/link ke video.
3. **Preview dan Notifikasi**:
   - Email ke `admin@internusa.com` dengan Excel, PDF, dan link video untuk review sebelum import ke Accurate.
   - Setelah approval, email ke `buyer@client.com` dengan PDF dan link video sebagai bukti.

### Contoh Output Preview
- **Excel/CSV**:
  ```
  Nomor_Mobil,Nama_Driver,Nomor_Stock_Opname,Detail_Barang,Timestamp,Foto_Muatan,Operator_Signature,Video_Link
  B 1234 XYZ,Budi Santoso,SO-2025-001,"50 box Mie ABC, batch #123, expiry 2026-01-01",2025-08-28 13:45:28,muatan_001.jpg,John Doe (digital sign),https://cloud.internusa.com/loading_001.mp4
  ```
- **PDF Bukti Muatan**:
  ```
  Bukti Muatan PT. Internusa Food
  Nomor_Mobil: B 1234 XYZ
  Nama_Driver: Budi Santoso
  Nomor_Stock_Opname: SO-2025-001
  Detail_Barang: 50 box Mie ABC, batch #123, expiry 2026-01-01
  Timestamp: 2025-08-28 13:45:28
  Foto_Muatan: muatan_001.jpg
  Operator_Signature: John Doe (digital sign)
  Video_CCTV: https://cloud.internusa.com/loading_001.mp4
  ```

### Contoh Script Python
Berikut snippet dari `main.py` untuk uji coba dengan CCTV:
```python
import pandas as pd
import datetime
from fpdf import FPDF
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders
import cv2
import os

# Data input (dari operator atau RFID)
data = {
    'Nomor_Mobil': 'B 1234 XYZ',
    'Nama_Driver': 'Budi Santoso',
    'Nomor_Stock_Opname': 'SO-2025-001',
    'Detail_Barang': '50 box Mie ABC, batch #123, expiry 2026-01-01',
    'Timestamp': datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
    'Foto_Muatan': 'muatan_001.jpg',
    'Operator_Signature': 'John Doe (digital sign)',
    'Video_Link': ''
}

# Rekam CCTV (trigger saat loading)
rtsp_url = 'rtsp://username:password@camera_ip:554/stream'  # Ganti dengan URL asli
cap = cv2.VideoCapture(rtsp_url)
fourcc = cv2.VideoWriter_fourcc(*'mp4v')
video_file = f'loading_{datetime.datetime.now().strftime("%Y%m%d_%H%M%S")}.mp4'
out = cv2.VideoWriter(video_file, fourcc, 20.0, (640, 480))
frame_count = 0
max_frames = 6000  # ~5 menit
while cap.isOpened() and frame_count < max_frames:
    ret, frame = cap.read()
    if not ret:
        break
    cv2.putText(frame, f'Stock Opname: SO-2025-001', (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)
    cv2.putText(frame, f'Timestamp: {datetime.datetime.now()}', (10, 60), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)
    out.write(frame)
    frame_count += 1
cap.release()
out.release()
cv2.destroyAllWindows()
data['Video_Link'] = f'https://cloud.internusa.com/{video_file}'  # Simulasi link cloud

# Simpan ke Excel
df = pd.DataFrame([data])
df.to_excel('outbound_data.xlsx', index=False)

# Generate PDF
class PDF(FPDF):
    def header(self):
        self.set_font('Arial', 'B', 12)
        self.cell(0, 10, 'Bukti Muatan PT. Internusa Food', 0, 1, 'C')

pdf = PDF()
pdf.add_page()
pdf.set_font('Arial', '', 10)
for key, value in data.items():
    pdf.cell(0, 10, f'{key}: {value}', 0, 1)
pdf.output('bukti_muatan.pdf')

# Kirim email preview
def send_email(to_email, subject, body, attachments):
    from_email = 'your_email@company.com'  # Ganti sesuai SMTP
    password = 'your_password'  # Ganti sesuai SMTP
    msg = MIMEMultipart()
    msg['From'] = from_email
    msg['To'] = to_email
    msg['Subject'] = subject
    msg.attach(MIMEText(body, 'plain'))
    for attachment in attachments:
        with open(attachment, 'rb') as f:
            part = MIMEBase('application', 'octet-stream')
            part.set_payload(f.read())
        encoders.encode_base64(part)
        part.add_header('Content-Disposition', f'attachment; filename={attachment}')
        msg.attach(part)
    server = smtplib.SMTP('smtp.gmail.com', 587)
    server.starttls()
    server.login(from_email, password)
    server.send_message(msg)
    server.quit()

send_email('admin@internusa.com', 'Preview Muatan dengan CCTV: SO-2025-001', 
           'Silakan review data dan video sebelum import ke Accurate.', 
           ['outbound_data.xlsx', 'bukti_muatan.pdf'])
send_email('buyer@client.com', 'Preview Bukti Pengiriman', 
           'Pesanan Anda dimuat. Lihat detail dan video di attachment.', 
           ['bukti_muatan.pdf'])
```

### Upgrade dengan RFID
1. Pasang RFID reader di loading dock.
2. Tambah script untuk baca tag RFID:
   ```python
   import serial
   ser = serial.Serial('COM3', 9600)  # Sesuaikan port
   rfid_id = ser.readline().decode().strip()
   data['RFID_ID'] = rfid_id
   ```

## Development Notes
- **Bahasa**: Python untuk backend, OpenCV untuk proses video CCTV, C++ (opsional) untuk performa tinggi via pybind11.
- **Hardware Tambahan**: IP camera dengan RTSP (contoh: Hikvision, Rp 10-50 juta per dock). Storage cloud untuk video (AWS S3/Google Cloud).
- **Testing**: Uji coba di 1 gudang dengan 1 kamera dan data dummy (1-2 minggu).
- **Ekspansi**: Integrasi API Accurate/WMS, tambah ML (PyTorch) untuk deteksi anomali di video CCTV atau prediksi delay pengiriman.

## Contributors
- **Mr. Don**: Pemilik proyek dan penyedia requirements.
- **Bejo**: Developer utama (ekspert Python, C++, dan sistem integrasi).

## License
MIT License. Bebas digunakan dan dimodifikasi untuk PT. Internusa Food.

## Contact
Hubungi Bejo via [email protected] untuk feedback atau revisi.
