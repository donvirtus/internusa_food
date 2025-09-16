# Gatekeeper (Django DTL) — PT. Internusa Food

Sistem Gatekeeper full-Django (tanpa React/Vite). UI dibangun dengan Django Template Language dan Tailwind CDN, fungsionalitas setara: login, dashboard, manajemen sesi loading, input pallet, dan penyelesaian sesi.

## Prasyarat

- Python 3.10+ (gunakan .venv yang sudah ada di `/home/donvirtus/hdd2/projects/.venv`)
- SQLite (bundled)

## Cara Menjalankan (Dev)

```bash
cd /home/donvirtus/hdd2/projects/pt_internusa
/home/donvirtus/hdd2/projects/.venv/bin/python manage.py migrate
/home/donvirtus/hdd2/projects/.venv/bin/python manage.py seed_demo
/home/donvirtus/hdd2/projects/.venv/bin/python manage.py runserver 0.0.0.0:8000
```

Buka: http://localhost:8000

## Kredensial Demo

- Admin: admin / securepass
- Operator: operator / password123

## Fitur

- Login/Logout berbasis session Django
- Dashboard: statistik (total, aktif, completed, today) + daftar sesi terbaru
- Sesi: list, create (truck plate, driver, SO, CCTV optional), detail (tambah/update pallet), complete session
- Admin site: /admin (kelola data)

## Struktur

```
pt_internusa/
  gatekeeper_project/      # settings, urls
  gatekeeper/              # app (models, views, admin, urls)
  templates/gatekeeper/    # DTL templates (base, login, dashboard, sessions/*)
  static/                  # static root (optional extra assets)
  media/                   # upload root (future use)
  manage.py
```

## Catatan

- Ini tidak bergantung pada proyek PaletteGuardian maupun React/Vite.
- Untuk production, gunakan WSGI server (gunicorn/uwsgi) dan atur STATIC/MEDIA melalui web server.


# Gatekeeper System for PT. Internusa Food

## 1. Gambaran Umum
Gatekeeper adalah sistem verifikasi dan pelacakan digital untuk PT. Internusa Food, ditempatkan di loading dock untuk memastikan palet dimuat ke truk sesuai Sales Order (SO) dari Accurate. Sistem ini menghilangkan ketidaksesuaian data, menyediakan bukti digital (foto, log, timestamp, video CCTV), dan menghasilkan laporan untuk import ke Accurate sebagai Delivery Order (DO).

- **Masalah yang Diatasi**:
  - Ketidaksesuaian jumlah barang vs SO (e.g., palet 48 kardus, SO butuh 40).
  - Kurangnya bukti digital untuk komplain pembeli.
  - Proses manual rawan error dan lambat.
- **Fokus Utama**: Lacak palet berdasarkan SO (detail barang ditangani WMS). Mulai dengan CSV/Excel, skalabel ke API nanti.
- **Manfaat**:
  - Kurangi komplain pembeli hingga 80% dengan bukti digital.
  - Tingkatkan efisiensi gudang via otomatisasi tracking.
- **Hardware**:
  - **RFID Reader**: EL-UHF-RC4-2 (TCP/IP, read-only, jarak 4-5m, EPC Gen2) untuk scan palet; HW-VX6336 (USB, read/write, jarak 0-20cm) untuk encode tag baru.
  - **Tag**: UHF RFID Metal Tag (Impinj Monza R6-P/NXP UCODE 8, anti-metal).
  - **CCTV**: Hikvision DS-2CD1021-I, Dahua DH-IPC-HDW1230T1-A-S5 (RTSP).
  - **Opsional**: OrangePi Zero 2 untuk MediaMTX (stream multi-camera).

## 2. Alur Kerja Sistem
1. **Persiapan Sesi Pemuatan**:
   - Operator buka Gatekeeper di browser (lokal: http://localhost:8000).
   - Input: Nomor plat truk (manual/OCR via CCTV), nama pengemudi, foto pengemudi (webcam), nomor SO (manual/CSV).
   - Sistem catat waktu mulai, trigger CCTV (Hikvision/Dahua).
2. **Pemindaian Palet**:
   - Import CSV WMS (barcode_id, item_type, quantity, batch, expiry).
   - Scan palet via EL-UHF-RC4-2 (TCP/IP) atau HW-VX6336 (USB, fallback).
   - Skenario Error:
     - **RFID Tag Belum Ada**: Encode baru via HW-VX6336, simpan mapping di DB.
     - **RFID Tag Rusak**: Encode tag baru via HW-VX6336.
     - **Barcode Rusak**: Input manual, encode RFID baru.
     - **Barcode dan RFID Rusak**: Input manual, encode RFID baru.
     - **Mismatch SO vs Palet**: Alert di UI, ganti palet.
     - **Interferensi RFID**: Scan ulang atau fallback ke HW-VX6336.
     - **CSV WMS Tidak Update**: Input manual sebagai "pending validation".
   - Validasi jumlah aktual dimuat vs SO, simpan sisa kardus di WMS.
3. **Penyelesaian Sesi**:
   - Upload foto muatan, simpan tautan CCTV (MP4 via OpenCV).
   - Tanda tangan digital (operator/driver) via canvas.
   - Status "Done", stop CCTV.
4. **Pelaporan**:
   - Dashboard admin: Review palet, foto, signature, video.
   - Ekspor CSV/Excel untuk DO Accurate, PDF dengan QR code ke video.

## 3. Tech Stack
- **Backend**: Django 4.2 (Python 3.12, Ubuntu 24.04).
- **Frontend**: Django Templates + TailwindCSS (via CDN).
- **Database**: SQLite (dev), PostgreSQL (produksi nanti).
- **Libraries**:
  - `opencv-python`: Rekam/stream RTSP CCTV (Hikvision/Dahua).
  - `pyserial`: Komunikasi USB HW-VX6336 (read/write).
  - `socket`: Komunikasi TCP/IP EL-UHF-RC4-2 (read).
  - `pandas`, `openpyxl`: Import/ekspor CSV/Excel.
  - `fpdf`: Generate PDF laporan.
  - `django-imagekit`: Resize foto/tanda tangan.
- **Hardware Integration**:
  - **EL-UHF-RC4-2**: TCP/IP (port 12345, contoh IP 192.168.1.100) untuk bulk scan palet.
  - **HW-VX6336**: USB Virtual Serial Port (COM di Windows, /dev/ttyUSB0 di Ubuntu) untuk encode tag.
  - **CCTV**: RTSP stream (e.g., rtsp://admin:password@<ip>:554/Streaming/Channels/101).
  - **OrangePi Zero 2**: MediaMTX untuk multi-stream (opsional, port 8554).

## 4. Requirements
- **Hardware**:
  - PC Ubuntu 24.04 (i3 12100/Ryzen 3 4350G/Ryzen 5 4600G, 32GB RAM).
  - RFID: EL-UHF-RC4-2 (TCP/IP), HW-VX6336 (USB), tag anti-metal.
  - CCTV: Hikvision DS-2CD1021-I, Dahua DH-IPC-HDW1230T1-A-S5.
  - OrangePi Zero 2 (opsional untuk MediaMTX).
- **Software**:
  - Python 3.12, pip, git.
  - VS Code untuk editing.
- **Lingkungan**: Lokal (dev), Replit free tier (testing online nanti).

## 5. Installation
1. **Setup Ubuntu**:
   ```bash
   sudo apt update
   sudo apt install python3-venv python3-dev libpq-dev build-essential
   python3 -m venv venv
   source venv/bin/activate
   pip install django==4.2 opencv-python==4.8.0 pyserial==3.5 pandas==2.0.3 openpyxl==3.1.2 fpdf==1.7.2 django-imagekit==5.0.0
