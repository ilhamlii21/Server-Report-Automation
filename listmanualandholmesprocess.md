# Pemetaan Metode Pengambilan Data Laporan Bulanan Server

Dokumen ini memetakan 11 poin utama dalam template laporan otomatisasi server bulanan Anda, lengkap dengan deskripsi fungsional serta metode pengambilannya (apakah menggunakan query langsung via n8n ke Prometheus/Grafana atau memerlukan analisis cerdas dari HolmesGPT menggunakan API Key).

---

## Daftar Pemetaan Poin Laporan

### 1. **Report Metadata**
* **Metode:** Manual n8n (Variabel internal & statis)
* **Deskripsi:** Berisi informasi dasar identitas laporan seperti Nama Server, Tanggal Pembuatan Laporan, Rentang Waktu Laporan (misal: 2 jam ke belakang atau 1 bulan lalu), dan alamat IP Server. Data ini diisi langsung menggunakan ekspresi bawaan n8n (seperti `$now`) atau variabel statis tanpa memproses database.

### 2. **Executive Summary**
* **Metode:** Analysis HolmesGPT + API Key
* **Deskripsi:** Rangkuman tingkat tinggi (naratif) mengenai kesehatan dan performa server selama periode laporan berjalan. AI (HolmesGPT) membaca metrik-metrik utama yang telah dikumpulkan, lalu menerjemahkannya ke dalam bahasa manusia yang formal dan mudah dipahami oleh tingkat manajemen/stakeholder.

### 3. **System Resource Overview**

* #### **3.1 CPU Usage**
  * **Metode:** Manual n8n ke Prometheus
  * **Deskripsi:** Menghitung persentase rata-rata (*Average*) dan puncak (*Peak*) penggunaan CPU server dengan mengeksekusi query PromQL (seperti `rate` dan `max_over_time`) ke Prometheus API melalui node HTTP Request n8n.
* #### **3.2 Memory + Swap**
  * **Metode:** Manual n8n ke Prometheus
  * **Deskripsi:** Menghitung total kapasitas RAM & Swap yang terpakai serta persentase sisa memori yang tersedia menggunakan metrik bawaan `node_memory_*`.
* #### **3.2 Top RAM Processes**
  * **Metode:** Manual n8n ke Prometheus (via process-exporter)
  * **Deskripsi:** Menampilkan daftar aplikasi teratas yang paling banyak memakan kapasitas memori RAM server menggunakan eksportir proses (`process-exporter`) dengan label pengelompokkan format `User:Aplikasi` (contoh: `ilhamvm2:python3`).
* #### **3.3 Disk Usage**
  * **Metode:** Manual n8n ke Prometheus
  * **Deskripsi:** Memantau kapasitas penyimpanan hardisk, persentase ruang terpakai, dan sisa kapasitas penyimpanan yang tersedia untuk partisi root (`/`) menggunakan metrik `node_filesystem_*`.
* #### **3.3 Top 5 Folder Terbesar**
  * **Metode:** Analysis HolmesGPT + API Key (atau Script Command)
  * **Deskripsi:** Mencari folder mana saja di direktori VM yang memakan ruang disk terbesar. Karena data ini tidak tersedia di exporter Prometheus standar, HolmesGPT bertugas masuk ke server secara terprogram, menjalankan perintah bash `du -sh` pada folder tertentu, dan menganalisis hasilnya.
* #### **3.4 Network Usage**
  * **Metode:** Manual n8n ke Prometheus
  * **Deskripsi:** Mengukur total lalu lintas data masuk (*Inbound / Receive*) dan keluar (*Outbound / Transmit*) pada kartu jaringan VM melalui metrik `node_network_*`.

### 4. **Service & Process Health**

* #### **4.1 Systemd Services**
  * **Metode:** Manual n8n ke Prometheus (via systemd-collector)
  * **Deskripsi:** Memantau status keaktifan layanan systemd utama (seperti NGINX, MySQL, Docker, n8n) dengan mengaktifkan fitur `systemd-collector` dan bypass keamanan AppArmor pada kontainer `node-exporter` host. Status dibaca berupa angka biner (`1` = Aktif, `0` = Mati).
* #### **4.2 PM2 Services**
  * **Metode:** Manual n8n ke Prometheus (via pm2-exporter)
  * **Deskripsi:** Memantau status keaktifan aplikasi berbasis Node.js di PM2 (seperti n8n) menggunakan metrik ril:
    * Status keaktifan: `pm2_status` (1 = online, 0 = stopped/errored)
    * Jumlah restart: `pm2_restarts`
    * Durasi berjalan: `time() - (pm2_uptime / 1000)` (detik)
* #### **4.3 Docker Health**
  * **Metode:** Manual n8n ke Prometheus (via cAdvisor)
  * **Deskripsi:** Memantau utilisasi resource (CPU/RAM) dan status kesehatan kontainer Docker yang aktif di VM menggunakan aplikasi **cAdvisor** yang terintegrasi dengan Prometheus.
* #### **4.4 Cronjob Audit**
  * **Metode:** Analysis HolmesGPT + API Key
  * **Deskripsi:** Mengaudit jalannya tugas terjadwal (cronjob) di server. HolmesGPT bertugas masuk ke server, menyisir file crontab (`crontab -l`, `/etc/crontab`), serta membaca log cron di syslog untuk menganalisis jika ada kegagalan eksekusi cronjob (*failing*).

### 5. **Security Audit**
* **Metode:** Analysis HolmesGPT + API Key
* **Deskripsi:** Mengaudit aspek keamanan sistem. HolmesGPT akan membaca log otentikasi sistem (`/var/log/auth.log`), menganalisis adanya percobaan login tidak sah (*brute force*), mendeteksi port terbuka yang tidak semestinya, serta membaca aturan firewall (`ufw`) aktif.

### 6. **Log Management**
* **Metode:** Analysis HolmesGPT + API Key
* **Deskripsi:** Menganalisis file log sistem (`/var/log/syslog`) secara kualitatif selama periode laporan untuk menemukan pesan anomali penting atau kegagalan kritis (seperti *Out of Memory*, disk error, atau crash aplikasi).

### 7. **Cost Monitoring**
* **Metode:** Manual n8n ke Azure Cost API
* **Deskripsi:** Memantau pengeluaran biaya infrastruktur VM Azure. n8n akan melakukan HTTP Request langsung ke API resmi **Azure Cost Management** untuk mengambil data tagihan real-time dalam format mata uang USD/IDR.

### 8. *(Poin 8 Kosong)*
* **Deskripsi:** Bagian ini kosong/dilewati pada template laporan asli.

### 9. **Change Log**
* **Metode:** Analysis HolmesGPT + API Key
* **Deskripsi:** Mencatat setiap perubahan konfigurasi pada server. HolmesGPT akan melacak riwayat instalasi/pembaruan paket aplikasi di sistem (`/var/log/dpkg.log`) serta mendeteksi modifikasi terbaru pada file konfigurasi utama seperti NGINX.

### 10. **Recommendations**
* **Metode:** Analysis HolmesGPT + API Key
* **Deskripsi:** Rekomendasi tindakan yang harus dilakukan admin sistem. HolmesGPT akan menganalisis keseluruhan temuan masalah pada server (seperti RAM hampir penuh, kapasitas disk sisa sedikit, cost naik, atau ada anomali keamanan), lalu merumuskan saran mitigasi/perbaikan dalam bentuk teks naratif.

### 11. **Attachments**
* **Metode:** Manual n8n ke Grafana Render API (Screenshot PNG)
* **Deskripsi:** Melampirkan bukti visual grafik resource dari dashboard Grafana. n8n akan melakukan request otomatis ke **Grafana Image Renderer API** untuk mengambil gambar (`.png`) tiap panel dashboard yang relevan sesuai rentang waktu laporan, lalu menyisipkannya ke dalam lampiran dokumen laporan.
