# Panduan Instalasi dan Dependensi Automasi Server Report (Workflow no Cronjob + Summary)

Dokumen ini berisi daftar lengkap aplikasi, exporter, dan dependensi sistem yang wajib diinstal pada server agar **Server Report Workflow no Cronjob + Summary** di n8n dapat berjalan dengan lancar. 

Workflow ini dirancang sepenuhnya menggunakan **HTTP Request** untuk mengambil data dari Prometheus API dan mengirimkan analisis ke AI API eksternal. Oleh karena itu, **tidak memerlukan node Execute Command (SSH)**, **tidak membutuhkan skrip cronjob lokal pada VM**, serta **tidak memerlukan instalasi Holmes AI lokal di server**. Semua kebutuhan AI dikelola langsung di dalam n8n menggunakan HTTP Request.

---

## 1. Fondasi Sistem & Containerization (Core Engine)

### A. Docker & Docker Compose
Digunakan untuk menjalankan Prometheus, Grafana, Grafana Image Renderer, cAdvisor, dan process-exporter dalam kontainer yang terisolasi.
* **Perintah Instalasi (Ubuntu/Debian):**
  ```bash
  sudo apt-get update
  sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common docker.io docker-compose
  sudo systemctl enable --now docker
  ```

### B. Node.js (v20) & npm
Runtime environment yang diperlukan untuk memasang n8n secara native dan modul PM2.
* **Perintah Instalasi:**
  ```bash
  curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
  sudo apt-get install -y nodejs
  ```

---

## 2. Orchestration & Automation (Server Runner)

### A. n8n (Self-hosted) & PM2
n8n bertindak sebagai orchestrator utama untuk mengumpulkan metrik, memanggil AI API untuk analisis, merender visualisasi Grafana, dan memperbarui Notion. PM2 mengelola proses n8n di latar belakang agar selalu aktif.
* **Perintah Instalasi:**
  ```bash
  # Instal n8n & PM2 secara global
  sudo npm install n8n pm2 -g

  # Jalankan n8n pertama kali via PM2 (menonaktifkan HTTPS secure cookie agar bisa diakses via HTTP biasa)
  NODES_EXCLUDE=[] N8N_SECURE_COOKIE=false pm2 start n8n

  # Konfigurasi auto-start PM2 saat booting VM
  pm2 startup
  # (Salin dan jalankan perintah keluaran yang diawali dengan sudo env...)
  
  # Simpan proses agar persisten
  pm2 save
  ```

---

## 3. Monitoring & Visualization Stack

Layanan-layanan di bawah ini dapat dideploy menggunakan file `docker-compose.yml` di dalam folder `~/monitoring`:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ~/monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    restart: unless-stopped

  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_RENDERING_SERVER_URL=http://grafana-renderer:8081/render
      - GF_RENDERING_CALLBACK_URL=http://grafana:3000/
      - GF_LOG_FILTERS=rendering:debug
    restart: unless-stopped

  grafana-renderer:
    image: grafana/grafana-image-renderer:latest
    container_name: grafana-renderer
    ports:
      - "8081"
    environment:
      - ENABLE_METRICS=true
      - AUTH_TOKEN=secret-renderer-token
    restart: unless-stopped
```

* **Grafana Image Renderer**: Berfungsi untuk mengambil tangkapan layar (screenshot) panel visualisasi grafik pada dashboard Grafana menjadi file gambar PNG untuk disisipkan ke Notion.

---

## 4. Prometheus Exporters (Metric Collectors)

Seluruh metrik performa server dikumpulkan menggunakan port-port exporter berikut:

### A. node_exporter (Port `9100`)
Mengumpulkan metrik hardware OS (CPU, RAM, Disk, Network) dan status Systemd Services.
* **Konfigurasi Docker Compose:**
  ```yaml
    node-exporter:
      image: prom/node-exporter:latest
      container_name: node-exporter
      security_opt:
        - apparmor:unconfined
      volumes:
        - /proc:/host/proc:ro
        - /sys:/host/sys:ro
        - /:/rootfs:ro
        - /var/run/dbus/system_bus_socket:/var/run/dbus/system_bus_socket:ro
      command:
        - '--path.procfs=/host/proc'
        - '--path.rootfs=/host/rootfs'
        - '--path.sysfs=/host/sys'
        - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
        - '--collector.systemd'
      restart: unless-stopped
  ```
  *(Catatan: flag `--collector.systemd` dan volume `system_bus_socket` diperlukan agar metrik status Nginx, Docker, dll dapat dibaca oleh Prometheus).*

### B. cAdvisor (Port `8080`)
Mengumpulkan metrik performa tiap container Docker secara real-time untuk data kesehatan Docker.
* **Perintah Jalankan Container:**
  ```bash
  docker run -d \
    --name=cadvisor \
    --volume=/:/rootfs:ro \
    --volume=/var/run:/var/run:ro \
    --volume=/sys:/sys:ro \
    --volume=/var/lib/docker/:/var/lib/docker:ro \
    --volume=/dev/disk/:/dev/disk:ro \
    --publish=8080:8080 \
    --device=/dev/kmsg \
    --privileged \
    --restart=unless-stopped \
    gcr.io/cadvisor/cadvisor:v0.47.0
  ```

### C. process-exporter (Port `9256`)
Memantau konsumsi RAM secara detail per kombinasi `User:Aplikasi` untuk mendeteksi proses pemakan memori terbesar.
* **Langkah Konfigurasi:**
  1. Buat folder dan file konfigurasi:
     ```bash
     sudo mkdir -p /opt/process-exporter
     sudo tee /opt/process-exporter/process-exporter.yml > /dev/null <<EOF
     process_names:
       - name: "{{.Username}}:{{.Comm}}"
         cmdline:
           - '.+'
     EOF
     ```
  2. Jalankan container:
     ```bash
     docker run -d \
       --name process-exporter \
       --restart unless-stopped \
       -p 9256:9256 \
       --privileged \
       -v /proc:/host/proc \
       -v /etc/passwd:/etc/passwd:ro \
       -v /opt/process-exporter:/config \
       ncabatoff/process-exporter \
       -procfs /host/proc \
       -config.path /config/process-exporter.yml
     ```

### D. pm2-prometheus-exporter (Port `9209`)
Modul PM2 untuk mengekspos data metrik internal aplikasi PM2 (list service, status online/stopped, restart count, uptime) ke Prometheus.
* **Perintah Instalasi:**
  ```bash
  pm2 install pm2-prometheus-exporter
  ```

---

## 5. Integrasi AI di n8n

Workflow ini **tidak memerlukan** instalasi Holmes AI lokal di VM. Semua analisis ringkasan performa (Summary) dan rekomendasi diproses menggunakan:
* **HTTP Request Node** di n8n yang memanggil AI API (`https://api.xxxxxx).
* Otentikasi dilakukan menggunakan Bearer Token (kredensial apikey) yang dikonfigurasikan di dalam n8n.
* Model yang digunakan di dalam payload request adalah model AI seperti `gpt-5.4` / `gpt-5.4-mini` untuk memproses data teks metrik yang telah dikumpulkan.

---

## 6. Status Ketersediaan Poin Laporan (Matriks Fungsionalitas)

Berikut adalah status poin laporan bulanan server berdasarkan arsitektur **tanpa skrip cronjob VM lokal** dan **tanpa Holmes AI**:

| No | Poin Laporan | Status | Keterangan / Solusi |
|---|---|---|---|
| **1** | **Report Metadata** | **✅ Bisa** | Diproses langsung menggunakan ekspresi bawaan n8n (`$now`) dan variabel statis. |
| **2** | **Executive Summary** | **🔄 Penyesuaian** | Berjalan di n8n via AI. Poin analisis disesuaikan hanya menggunakan metrik yang aktif (mengabaikan data dari poin yang gagal). |
| **3** | **System Resource Overview** | | |
| | - 3.1. CPU Usage | **✅ Bisa** | Menggunakan metrik standard `node_cpu_seconds_total` dari `node_exporter`. |
| | - 3.2. RAM Usage | **✅ Bisa** | Menggunakan metrik `node_memory_Mem*` dan `process-exporter` untuk visualisasi RAM per grup aplikasi. |
| | - 3.3. Disk (Top 5 Folder Terbesar) | **❌ Tidak Bisa** | `node_exporter` standard tidak dapat melakukan pemindaian ukuran folder (`du`) secara bawaan tanpa skrip cronjob lokal. |
| | - 3.4. Network Bandwidth | **✅ Bisa** | Menggunakan metrik `node_network_*` dari `node_exporter`. |
| **4** | **Service & Process Health** | | |
| | - 4.1. Systemd Services | **✅ Bisa** | Mengaktifkan flag `--collector.systemd` pada container `node_exporter` untuk memantau status service VM. |
| | - 4.2. PM2 Services | **✅ Bisa** | Menggunakan metrik dari `pm2-prometheus-exporter`. |
| | - 4.3. Docker Health | **✅ Bisa** | Menggunakan metrik kontainer real-time yang diekspos oleh **cAdvisor**. |
| | - 4.4. Cronjob Health | **❌ Tidak Bisa** | Tidak ada skrip lokal untuk membaca log crontab atau syslog cron, sehingga metrik kegagalan cron tidak tersedia. |
| **5** | **Security Audit** | **❌ Tidak Bisa** | Memeriksa log auth SSH dan konfigurasi firewall membutuhkan pembacaan file sistem aman (`/var/log/auth.log` & `ufw status`) yang tidak diekspos oleh exporter standar tanpa tool pembantu lokal. |
| **6** | **Log Management** | **❌ Tidak Bisa** | Pemindaian pesan anomali/error secara kualitatif pada `/var/log/syslog` tidak dapat dilakukan tanpa agent log lokal. |
| **7** | **Cost Monitoring** | **-** | Menggunakan integrasi API eksternal (misal Azure Cost API) via HTTP Request n8n jika diperlukan. |
| **9** | **Change Log** | **❌ Tidak Bisa** | Pelacakan perubahan instalasi package (`/var/log/dpkg.log`) dan modifikasi konfigurasi Nginx membutuhkan akses file lokal. |
| **10**| **Recommendation** | **🔄 Penyesuaian** | Berjalan di n8n via AI. Poin saran disesuaikan hanya menggunakan metrik yang berhasil dikumpulkan (CPU, RAM, PM2, Docker, Systemd). |
| **11**| **Attachments** | | |
| | - 11.1. Grafana Render Image | **✅ Bisa** | Berhasil mengambil gambar grafik PNG via API **Grafana Image Renderer**. |
| | - 11.2. Analisis AI | **✅ Bisa** | Hasil generate analisis dari model AI di Notion. |


