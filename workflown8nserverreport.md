# Dokumentasi Alur Kerja n8n: Server Report Workflow

Dokumen ini menjelaskan arsitektur, detail node, dan fungsi dari workflow n8n yang digunakan untuk mengotomatisasi pembuatan **Laporan Bulanan Performa dan Kesehatan Server** ke Notion.

---

## 1. Ringkasan Alur Kerja (Workflow Overview)

Workflow ini dirancang untuk:
1. **Mengambil metrics performa** (CPU, RAM, Swap, Disk, Network) dari server menggunakan HTTP Request ke **Prometheus API**.
2. **Memeriksa kesehatan layanan** (Systemd, PM2, Docker Container, Cronjob) dengan menjalankan perintah CLI secara lokal via node **Execute Command**.
3. **Melakukan audit keamanan dan perubahan** (SSH logins, package update, file config nginx, docker image, systemd service) melalui asisten **Holmes AI CLI**.
4. **Merender grafik performa** dari **Grafana** (mengubah dashboard panel menjadi file `.png`).
5. **Mengirim & memperbarui data** tersebut secara terstruktur langsung ke halaman **Notion** menggunakan Notion API.

---

## 2. Struktur Poin Laporan (Notion Blocks Mapping)

Workflow ini dibagi menjadi beberapa modul visual (menggunakan *Sticky Notes*) yang mewakili poin-poin dalam template Laporan Server di Notion:

### **Poin 3: Sumber Daya Server (Server Resources)**
Bagian ini mengambil data performa server menggunakan **HTTP Request ke Prometheus API** (tanpa CLI Command di n8n):
* **3.1. CPU Usage**:
  - `Prometheus AVG CPU`: Mengambil rata-rata utilisasi CPU selama 2 jam terakhir (HTTP GET ke Prometheus).
  - `Prometheus PEAK CPU`: Mengambil puncak (peak) utilisasi CPU selama 2 jam terakhir (HTTP GET ke Prometheus).
  - `Holmes ask CPU` / `AI Analysis`: Menganalisis data utilisasi CPU untuk memberikan catatan rekomendasi.
  - **Notion Block**: Memperbarui bullet list dengan Avg CPU, Peak CPU, dan rekomendasi AI.
* **3.2. RAM & Swap Usage**:
  - `Prometheus AVG RAM` & `Prometheus Peak RAM`: Mengambil rata-rata dan peak RAM usage via Prometheus API.
  - `Prometheus Swap Used` & `Prometheus Swap Total`: Mengambil kapasitas swap terpakai dan total swap via Prometheus API.
  - `Top 5 Proses Makan RAM`: Mengambil data 5 proses teratas yang mengonsumsi memori terbesar menggunakan query Prometheus (`topk(5, node_process_memory_bytes)` atau sejenisnya) tanpa menjalankan command SSH langsung.
  - **Notion Block**: Memperbarui data RAM, Swap, top processes, dan rekomendasi RAM.
* **3.3. Disk Usage**:
  - `Prometheus Disk Usage`: Mengambil persentase ruang disk terpakai pada partisi root (`/`) via Prometheus API.
  - `Prometheus Pertumbuhan Disk`: Mengambil pertumbuhan disk dibanding 30 hari yang lalu via Prometheus API.
  - `Top 5 Folder Terbesar` (Penting): Tidak lagi dijalankan langsung melalui CLI n8n, melainkan menggunakan **Cronjob lokal VM** (`collect_top_files.sh`) yang mencatat kapasitas 5 file/folder terbesar di `/var/log/*` ke dalam file `.prom` untuk dibaca oleh `node_exporter` Textfile Collector. n8n kemudian memanggil data ini via Prometheus query `topk(5, node_dir_file_size_bytes)`.
  - **Notion Block**: Mengirim request PATCH HTTP langsung ke API Notion untuk menyisipkan list info disk dan block code berisi 5 folder log terbesar.
* **3.4. Network Bandwidth**:
  - `Prometheus NET Inbound` & `Prometheus NET Outbond`: Mengambil rata-rata traffic masuk (RX) dan keluar (TX) dalam Mbps menggunakan query Prometheus.
  - **Notion Block**: Mengirim data rata-rata RX/TX dan catatan jaringan ke Notion.

---

### **Poin 4: Status Layanan & Aplikasi (Services & Apps Status)**
Bagian ini memantau service yang berjalan di server:
* **4.1. Systemd Services & Zombie Processes**:
  - `Active/Failed`: Prometheus query untuk mendeteksi unit systemd yang berstatus `failed`.
  - `Zombie Process`: Prometheus query untuk mendeteksi jumlah zombie process.
  - `Notes : Systemd Holmes ask`: Evaluasi formal dari Holmes mengenai status systemd.
  - **Notion Block**: Menuliskan daftar service yang gagal dan total zombie process.
* **PM2 Services (Node.js)**:
  - `Prometheus PM2 List services` & `Prometheus PM2 Restart`: Mengambil status berjalan (`online`/`stopped`) dan total restart dari services yang dikelola PM2.
  - `Prometheus PM2 Restart1`: Mengambil metrik uptime service PM2.
  - `Notes PM2`: Rekomendasi/notes performa PM2.
  - **Notion Block**: Menyisipkan daftar service, jumlah restart, uptime (dikonversi dari detik ke hari), dan notes ke Notion.
* **Docker Health**:
  - `Docker Health / Storage`: Metrik kesehatan Docker (volume count/size, image count/size, reclaimable space) dan jumlah kontainer berjalan dikumpulkan secara lokal di VM via cronjob (`collect_docker_storage.sh`) ke node_exporter Textfile Collector, lalu dipanggil oleh n8n via Prometheus API HTTP Request.
  - `Keterbatasan cAdvisor`: cAdvisor/Prometheus hanya bisa menampilkan jumlah total kontainer berjalan (`node_docker_containers_running`) dan data kapasitas disk Docker. Tidak bisa get nama kontainer + uptime dinamis di Notion secara langsung karena keterbatasan format matriks Prometheus.
  - **Notion Block**: Memperbarui status kesehatan Docker di Notion secara berkala.
* **Cronjob Status**:
  - `Cronjob`: Memeriksa kegagalan eksekusi cronjob lokal VM menggunakan data logs.

---

### **Poin 5: Audit Keamanan (Security Audit)**
* **Security Audit Node & JavaScript Parser**:
  - Menjalankan analisis keamanan dasar pada VM.
  - **Notion Block**: Mengirimkan status login SSH, deteksi IP mencurigakan, update package, masa berlaku SSL, dan status firewall (UFW) ke Notion.

---

### **Poin 9: Log Perubahan VM (Change Log)**
* `Log Management Holmes`:
  - Holmes menjalankan perintah audit sistem selama 30 hari terakhir.
  - Memeriksa paket ter-update (`/var/log/dpkg.log`), OS update, konfigurasi Nginx, cronjob baru, firewall rule (UFW), docker image baru, dan status service systemd.
  - Mengembalikan output berformat JSON valid.

---

### **Poin 10: Ringkasan Eksekutif & Rekomendasi**
* `Recommendations for next month`:
  - Mengirim seluruh data JSON performa yang dikumpulkan ke Holmes.
  - Holmes bertindak sebagai asisten DevOps untuk menghasilkan 1 paragraf **Executive Summary** formal dan **3-5 poin rekomendasi** perbaikan konkret untuk bulan berikutnya dalam Bahasa Indonesia.

---

### **Poin 11: Lampiran Grafik (Grafana Renders)**
* **Grafana Panel Image Rendering & Supabase Integration**:
  - `Get Image CPU`, `Get Image RAM`, `Get Image DISK`, `Get Image APPS RAM`:
    Mengunduh gambar grafik format PNG langsung dari Grafana Image Renderer via HTTP GET (Response Format: File/Binary).
  - `HTTP Supabase Bucket (1/2/3)`:
    Mengunggah secara mandiri file grafik PNG tersebut ke **Supabase Storage** (bucket `grafanarenderiimage`) menggunakan HTTP POST + `service_role` key.
  - `MinIO Status`:
    Sempat dicoba menggunakan MinIO lokal VM (S3-compatible) dengan membuat bucket `imagerenderminio` dan akses `public/download` via CLI `mc`, namun integrasi di n8n masih terhambat masalah otentikasi browser autofill yang menyisipkan sandi salah ke credential n8n (akan dikerjakan lebih lanjut).
  - `Merge` & `Notion Node`:
    Menggabungkan output upload dan menyisipkan 4 grafik dari Supabase ke dalam Notion sekaligus menggunakan block image.

---

## 3. Detail Integrasi Notion API (HTTP Request Nodes)

Untuk beberapa poin seperti Disk Usage (3.3) dan PM2 (3.x), workflow ini tidak menggunakan modul bawaan Notion n8n, melainkan node **HTTP Request** native untuk memanggil Notion API secara langsung via metode `PATCH`.

### Contoh Konfigurasi HTTP Request:
* **Endpoint**: `https://api.notion.com/v1/blocks/{BLOCK_ID}/children`
* **Method**: `PATCH`
* **Headers**:
  - `Authorization`: `Bearer ntn_6726671...`
  - `Notion-Version`: `2022-06-28`
  - `Content-Type`: `application/json`
* **JSON Body (Dynamic)**:
  ```json
  {
    "children": [
      {
        "object": "block",
        "type": "bulleted_list_item",
        "bulleted_list_item": {
          "rich_text": [
            {
              "type": "text",
              "text": {
                "content": "Disk % per partition: {{ ... }}"
              }
            }
          ]
        }
      }
    ]
  }
  ```

---

## 4. Cara Mengimpor Workflow di n8n

Jika Anda ingin memulihkan atau menduplikasi workflow ini di instance n8n baru:
1. Salin seluruh isi file JSON **[Server Report Workflow (6).json](file:///d:/Timedoor/Task/Server%20Report%20by%20Render%20Image/Documentation/Server%20Report%20Workflow%20(6).json)**.
2. Buka dashboard n8n Anda.
3. Buat workflow baru.
4. Klik area canvas kosong, lalu tekan tombol **Ctrl + V** (atau klik menu di kanan atas dan pilih **Import from file/JSON**).
5. Sesuaikan **Notion Credentials** dan pastikan **Grafana Auth Token** serta **Prometheus URL** (`http://20.205.17.18:9090`) dapat diakses dari instance n8n tersebut.
