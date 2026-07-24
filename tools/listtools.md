# Daftar Lengkap Tools, Exporter, dan Dependensi Automasi Server Report

Dokumen ini berisi daftar lengkap aplikasi, tools, exporter, dan dependensi sistem yang dibutuhkan untuk melakukan migrasi ke server baru berdasarkan dokumentasi penelitian automasi laporan bulanan. Dokumen ini dilengkapi dengan deskripsi fungsi serta perintah Linux (`command`) untuk proses instalasi dan konfigurasinya.

---

## 1. Fondasi Sistem & Containerization (Core Engine)

### A. Docker & Docker Compose
* **Fungsi:** Untuk menjalankan kontainer Prometheus, Grafana Image Renderer, cAdvisor, dan Process Exporter secara terisolasi.
* **Perintah Linux (Instalasi):**
  ```bash
  # Perbarui indeks paket lokal
  sudo apt-get update

  # Instal dependensi yang diperlukan untuk HTTPS repository
  sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

  # Instal Docker dan Docker Compose Utility
  sudo apt-get install -y docker.io docker-compose
  
  # Aktifkan dan jalankan Docker daemon
  sudo systemctl enable --now docker
  ```

### B. Node.js (v20) & npm
* **Fungsi:** Runtime environment utama yang dibutuhkan untuk menjalankan n8n secara self-hosted dan instalasi package global.
* **Dokumentasi Referensi:** [n8nselfhost.md](file:///d:/Timedoor/Task/Server%20Report%20by%20Render%20Image/Documentation/n8nselfhost.md)
* **Perintah Linux (Instalasi):**
  ```bash
  # Menambahkan repository NodeSource untuk Node.js v20
  curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

  # Menginstal Node.js dan npm
  sudo apt-get install -y nodejs

  # Verifikasi instalasi runtime
  node -v
  npm -v
  ```

---

## 2. Orchestration & Automation (Server Runner)

### A. n8n (Self-hosted via PM2/Node.js)
* **Fungsi:** Orchestrator utama untuk memicu cron, mengambil screenshot Grafana, mengambil data Prometheus, memformat data, dan mengirimkannya ke Notion/Slack/Email.
* **Dokumentasi Referensi:** [n8nselfhost.md](file:///d:/Timedoor/Task/Server%20Report%20by%20Render%20Image/Documentation/n8nselfhost.md)
* **Perintah Linux (Instalasi):**
  ```bash
  # Instal n8n secara global di sistem menggunakan npm
  sudo npm install n8n -g

  # Verifikasi instalasi n8n
  n8n -v
  ```

### B. PM2 (Process Manager)
* **Fungsi:** Mengelola proses n8n di latar belakang (background), menangani auto-start saat booting VM, serta memastikan layanan tetap berjalan.
* **Dokumentasi Referensi:** [n8nselfhost.md](file:///d:/Timedoor/Task/Server%20Report%20by%20Render%20Image/Documentation/n8nselfhost.md)
* **Perintah Linux (Instalasi & Konfigurasi):**
  ```bash
  # 1. Instal PM2 secara global
  sudo npm install pm2 -g

  # 2. Jalankan n8n pertama kali menggunakan PM2 dengan variabel lingkungan khusus
  NODES_EXCLUDE=[] N8N_SECURE_COOKIE=false pm2 start n8n

  # 3. Konfigurasikan Auto-Start saat VM Booting
  pm2 startup
  # (Salin dan jalankan perintah keluaran yang dihasilkan dari terminal Anda, yang diawali dengan sudo env...)

  # 4. Simpan konfigurasi proses PM2 agar persisten
  pm2 save
  ```

---

## 3. Monitoring & Visualization Stack

### A. Grafana
* **Fungsi:** Dashboard visualisasi metrik server dan penyedia API untuk panel rendering gambar.
* **Perintah Linux (Docker Deployment):**
  Layanan ini dideploy melalui Docker. Contoh entri pada `docker-compose.yml`:
  ```yaml
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
  ```

### B. Grafana Image Renderer (Docker Container)
* **Fungsi:** Layanan khusus untuk mengubah panel dashboard Grafana menjadi file gambar (PNG) via Render API n8n.
* **Perintah Linux (Docker Deployment):**
  Kontainer tersendiri yang terintegrasi dengan Grafana. Contoh konfigurasi pada `docker-compose.yml`:
  ```yaml
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

### C. Prometheus
* **Fungsi:** Database time-series utama untuk menyimpan data metrik sistem dan mengeksekusi query PromQL dari n8n.
* **Perintah Linux (Docker Deployment):**
  Dideploy via Docker Compose. Contoh konfigurasi pada `docker-compose.yml`:
  ```yaml
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ~/monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    restart: unless-stopped
  ```

---

## 4. Prometheus Exporters (Tier 2 - Metrics Gathering)

### A. node_exporter (dengan Collector Systemd Enabled)
* **Fungsi:** Mengumpulkan metrik OS (CPU, Memory, Disk, Network) dan dipasang dengan mounting socket D-Bus host VM + AppArmor profile unconfined untuk melacak status kesehatan Systemd Services (seperti status Nginx/Docker).
* **Dokumentasi Referensi:** [systemdservices.md](file:///d:/Timedoor/Task/Server%20Report%20by%20Render%20Image/Documentation/systemdservices.md)
* **Perintah Linux (Docker Deployment):**
  Tambahkan konfigurasi berikut pada file `docker-compose.yml` di server baru Anda:
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
  Jalankan perintah berikut untuk mengaplikasikan konfigurasi:
  ```bash
  docker compose up -d node-exporter
  ```

### B. cAdvisor (Container Advisor)
* **Fungsi:** Mengumpulkan metrik performa dan penggunaan resource khusus dari setiap Docker Container yang berjalan.
* **Perintah Linux (Docker Run / Compose):**
  Untuk menjalankan kontainer cAdvisor di server baru:
  ```bash
  docker run -d \
    --volume=/:/rootfs:ro \
    --volume=/var/run:/var/run:ro \
    --volume=/sys:/sys:ro \
    --volume=/var/lib/docker/:/var/lib/docker:ro \
    --volume=/dev/disk/:/dev/disk:ro \
    --publish=8080:8080 \
    --detach=true \
    --name=cadvisor \
    --privileged \
    --device=/dev/kmsg \
    --restart=unless-stopped \
    gcr.io/cadvisor/cadvisor:v0.47.0
  ```

### C. process-exporter (Docker Container)
* **Fungsi:** Memantau penggunaan memori RAM secara detail berdasarkan klasifikasi User:Aplikasi (misal: root:mysqld).
* **Dokumentasi Referensi:** [processexporter.md](file:///d:/Timedoor/Task/Server%20Report%20by%20Render%20Image/Documentation/processexporter.md)
* **Perintah Linux (Instalasi & Deployment):**
  ```bash
  # 1. Buat direktori konfigurasi di host VM baru
  sudo mkdir -p /opt/process-exporter

  # 2. Buat file konfigurasinya
  sudo tee /opt/process-exporter/process-exporter.yml > /dev/null <<EOF
  process_names:
    - name: "{{.Username}}:{{.Comm}}"
      cmdline:
        - '.+'
  EOF

  # 3. Jalankan kontainer process-exporter
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

### D. pm2-prometheus-exporter
* **Fungsi:** Modul PM2 (diakses di port 9209) untuk mengekspos metrik internal PM2 (uptime, restart count/crash, memori n8n) ke Prometheus.
* **Dokumentasi Referensi:** [pm2eksporter.md](file:///d:/Timedoor/Task/Server%20Report%20by%20Render%20Image/Documentation/pm2eksporter.md)
* **Perintah Linux (Instalasi):**
  ```bash
  # Menginstal modul exporter secara langsung di PM2
  pm2 install pm2-prometheus-exporter
  ```

---

## 5. Logging, Security, & Custom Metrics (Tier 3 - Advanced Setup)

### A. Promtail & Grafana Loki
* **Fungsi:** Log aggregation system untuk memantau log sistem lokal (misal: log SSH /var/log/auth.log untuk kebutuhan audit keamanan).
* **Perintah Linux (Docker Deployment):**
  Dideploy menggunakan Docker Compose. Contoh konfigurasi pada `docker-compose.yml`:
  ```yaml
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - /var/log:/var/log
      - ~/monitoring/promtail-config.yaml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    restart: unless-stopped
  ```

### B. blackbox_exporter
* **Fungsi:** Memantau status kedaluwarsa sertifikat SSL HTTPS dan ketersediaan HTTP endpoint.
* **Perintah Linux (Docker Deployment):**
  ```bash
  docker run -d \
    --name blackbox_exporter \
    -p 9115:9115 \
    --restart unless-stopped \
    prom/blackbox-exporter:latest
  ```

### C. Prometheus Pushgateway
* **Fungsi:** Menampung metrik hasil script custom (seperti audit crontab atau script du -sh untuk folder size) sebelum ditarik oleh Prometheus.
* **Perintah Linux (Docker Deployment):**
  ```bash
  docker run -d \
    --name pushgateway \
    -p 9091:9091 \
    --restart unless-stopped \
    prom/pushgateway:latest
  ```

---

## 6. System Utility & Command-Line Tools

### A. curl & jq
* **Fungsi:** Menguji koneksi API dan melakukan parsing data JSON pada terminal atau node Execute Command.
* **Perintah Linux (Instalasi):**
  ```bash
  sudo apt-get install -y curl jq
  ```

### B. wkhtmltopdf (opsional, diinstal di sistem host)
* **Fungsi:** Digunakan jika Anda ingin melakukan konversi template laporan HTML menjadi file PDF secara lokal via command line di n8n.
* **Perintah Linux (Instalasi):**
  ```bash
  sudo apt-get install -y wkhtmltopdf
  ```
