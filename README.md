# Gatekeeper System for PT. Internusa Food

## Overview
Gatekeeper adalah sistem modular berbasis software yang dirancang untuk PT. Internusa Food guna mengatasi masalah ketidaksinkronan data pasca-barang keluar gudang menuju pengiriman ke buyer. Sistem ini berfungsi sebagai "penjaga gerbang" di loading dock, memverifikasi dan mencatat detail muatan palet ke kendaraan pengangkut berdasarkan sales order (SO), menghasilkan dokumen laporan untuk review sebelum import ke Accurate, dan menyediakan bukti digital (foto dan video CCTV) untuk menangani komplain buyer. Fokusnya adalah pada pelacakan palet, karena detail barang individual (jumlah, batch, dll.) sudah ditangani oleh tim produksi.

**Tujuan Utama:**
- Tracking akurat: Mengetahui palet apa yang dimuat ke truk mana berdasarkan nomor sales order.
- Preview data: Menghasilkan laporan untuk review manual sebelum import ke Accurate (surat jalan).
- Bukti anti-komplain: Memberikan dokumentasi digital (foto, log, timestamp, dan video CCTV) untuk buyer.
- Pendekatan bertahap: Uji coba via Excel tanpa ganggu sistem existing (Accurate/WMS).

**Masalah yang Diatasi:**
- Ketidaksinkronan stok antara Accurate (agregat) dan WMS (real-time).
- Kurangnya bukti visual/digital, menyebabkan komplain buyer soal palet yang kurang/tidak sesuai pesanan.
- Proses manual yang rentan error.

**Manfaat:**
- Kurangi komplain buyer hingga 80% dengan bukti digital dan video.
- Tingkatkan efisiensi operasional gudang dengan fokus pada pelacakan palet.
- Skalabel untuk integrasi API atau RFID penuh di masa depan.

## Features
### Fitur Utama
- **Tracking Muatan**:
  - Catat: Nomor mobil pengangkut (contoh: B 1234 XYZ), nama driver, nomor sales order (SO-2025-001), Palet_ID (contoh: PLT-001).
  - Tambahan: Timestamp loading, foto muatan palet, signature digital operator, dan rekaman CCTV selama proses loading.
- **Pembuatan Dokumen Laporan**:
  - Excel/CSV file untuk preview data muatan palet sebelum import ke Accurate.
  - PDF bukti muatan dengan foto, log Palet_ID, dan link ke rekaman CCTV.
- **Notifikasi Otomatis**:
  - Email/SMS ke admin internal untuk review data (dengan Excel, PDF, dan link video).
  - Email/SMS ke buyer dengan bukti digital setelah approval.
- **Integrasi CCTV**:
  - Rekam proses loading via IP camera (RTSP stream) di loading dock.
  - Auto-overlay timestamp dan nomor sales order/Palet_ID pada video.
  - Simpan klip MP4, link ke output preview untuk review.
- **Integrasi RFID**:
  - Scan otomatis tag RFID yang ditempel pada palet untuk validasi real-time sesuai sales order.
- **Dashboard Monitoring (Opsional)**:
  - Web-based view untuk status muatan palet dan playback video CCTV.

### Fitur Keamanan
- Enkripsi AES untuk log, bukti digital, dan metadata video.
- Role-based access: Operator hanya input, admin approve/export.
- Audit trail dengan timestamp untuk setiap aksi, termasuk akses video.

## Requirements
- **Hardware**:
  - Laptop/PC untuk operator gudang.
  - Ponsel dengan kamera untuk foto muatan palet.
  - IP camera (contoh: Hikvision) dengan RTSP support untuk rekaman CCTV.
  - RFID reader (handheld/fixed) dan tag RFID pasif untuk setiap palet.
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
     - Port RFID reader untuk palet.
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
     - Nomor mobil, nama driver, nomor sales order, Palet_ID.
     - Timestamp (otomatis), foto muatan palet (upload), signature digital.
     - Rekaman CCTV otomatis di-trigger saat proses loading dimulai.
2. **Proses Data**:
   - Script generate `outbound_data.xlsx` untuk preview (termasuk link ke video CCTV).
   - Generate `bukti_muatan.pdf` dengan detail palet, foto, dan QR code/link ke video.
3. **Preview dan Notifikasi**:
   - Email ke `admin@internusa.com` dengan Excel, PDF, and link video untuk review sebelum import ke Accurate.
   - Setelah approval, email ke `buyer@client.com` dengan PDF dan link video sebagai bukti.

### Contoh Output Preview
- **Excel/CSV**:
  ```
  Nomor_Mobil,Nama_Driver,Nomor_Sales_Order,Palet_ID,Timestamp,Foto_Muatan,Operator_Signature,Video_Link
  B 1234 XYZ,Budi Santoso,SO-2025-001,PLT-001,2025-08-28 15:53:28,muatan_001.jpg,John Doe (digital sign),https://cloud.internusa.com/loading_001.mp4
  ```
- **PDF Bukti Muatan**:
  ```
  Bukti Muatan PT. Internusa Food
  Nomor_Mobil: B 1234 XYZ
  Nama_Driver: Budi Santoso
  Nomor_Sales_Order: SO-2025-001
  Palet_ID: PLT-001
  Timestamp: 2025-08-28 15:53:28
  Foto_Muatan: muatan_001.jpg
  Operator_Signature: John Doe (digital sign)
  Video_CCTV: https://cloud.internusa.com/loading_001.mp4
  ```

### Contoh Script Python
Berikut snippet dari `main.py` untuk uji coba dengan CCTV dan RFID untuk palet:
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
import serial

# Data input (dari operator atau RFID)
data = {
    'Nomor_Mobil': 'B 1234 XYZ',
    'Nama_Driver': 'Budi Santoso',
    'Nomor_Sales_Order': 'SO-2025-001',
    'Palet_ID': 'PLT-001',
    'Timestamp': datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
    'Foto_Muatan': 'muatan_001.jpg',
    'Operator_Signature': 'John Doe (digital sign)',
    'Video_Link': ''
}

# Scan RFID untuk Palet_ID
ser = serial.Serial('COM3', 9600)  # Sesuaikan port RFID reader
rfid_id = ser.readline().decode().strip()
data['Palet_ID'] = rfid_id if rfid_id else data['Palet_ID']  # Fallback ke input manual

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
    cv2.putText(frame, f'Sales Order: SO-2025-001', (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)
    cv2.putText(frame, f'Palet_ID: {data["Palet_ID"]}', (10, 60), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)
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
           'Silakan review data palet dan video sebelum import ke Accurate.', 
           ['outbound_data.xlsx', 'bukti_muatan.pdf'])
send_email('buyer@client.com', 'Preview Bukti Pengiriman', 
           'Pesanan Anda dimuat. Lihat detail palet dan video di attachment.', 
           ['bukti_muatan.pdf'])
```

### Upgrade dengan RFID
1. Pasang tag RFID pasif di setiap palet.
2. Pasang RFID reader di loading dock untuk scan otomatis saat palet dimuat.
3. Script di atas sudah include baca RFID untuk update Palet_ID.

## Development Notes
- **Bahasa**: Python untuk backend, OpenCV untuk proses video CCTV, C++ (opsional) untuk performa tinggi via pybind11.
- **Catatan Sales Order dan Palet**: Nomor sales order (SO) merujuk ke pesanan penjualan, dan Palet_ID adalah identitas unik palet yang mewadahi karton produk. Detail barang (jumlah, batch, expiry) diurus tim produksi, sehingga Gatekeeper fokus pada pelacakan palet.
- **Hardware Tambahan**: IP camera dengan RTSP (contoh: Hikvision, Rp 10-50 juta per dock), RFID reader (Rp 5-20 juta per unit), tag RFID pasif (Rp 500-2000 per palet). Storage cloud untuk video (AWS S3/Google Cloud).
- **Testing**: Uji coba di 1 gudang dengan 1 kamera, 1 RFID reader, dan data dummy (1-2 minggu).
- **Ekspansi**: Integrasi API Accurate/WMS setelah uji coba sukses, tambah ML (PyTorch) untuk deteksi anomali di video CCTV atau prediksi delay pengiriman.

## Contributors
- **Mr. Don**: Pemilik proyek dan penyedia requirements.
- **Bejo**: Developer utama (ekspert Python, C++, dan sistem integrasi).

## License
MIT License. Bebas digunakan dan dimodifikasi untuk PT. Internusa Food.

## Contact
Hubungi Bejo via [email protected] untuk feedback atau revisi.
