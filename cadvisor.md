# Panduan Konfigurasi Pelacakan Container Docker dengan cAdvisor dan Prometheus

Dokumen ini menjelaskan langkah-langkah untuk memasang **cAdvisor** (Container Advisor) di server, mengintegrasikannya dengan **Prometheus**, dan memantau status kesehatan kontainer Docker secara langsung menggunakan metrik cAdvisor melalui HTTP Request.

---

## 1. Pendahuluan
Secara bawaan, `node-exporter` hanya memantau metrik tingkat sistem operasi host (CPU, RAM, Disk, Jaringan). Untuk memantau metrik spesifik kontainer Docker (seperti status running, jumlah restart, penggunaan CPU/RAM per container) secara aman tanpa membuka port Docker API ke luar, kita menggunakan **cAdvisor** dari Google.

cAdvisor berjalan sebagai kontainer terisolasi yang membaca status kontainer lain secara langsung dari subsystem cgroups Linux.

---

## 2. Langkah 1: Pemasangan cAdvisor (Docker / Docker Compose)

### Opsi A: Menggunakan Docker Compose (Sangat Direkomendasikan)
Tambahkan blok konfigurasi `cadvisor` di bawah bagian `services` pada file `docker-compose.yml` Anda (misalnya di `~/monitoring/docker-compose.yml`):

```yaml
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.0
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    devices:
      - /dev/kmsg
    privileged: true
    restart: unless-stopped
```

Jalankan perintah berikut untuk mengunduh dan mengaktifkan kontainer:
```bash
cd ~/monitoring
docker compose up -d cadvisor
```

### Opsi B: Menggunakan Docker Run (CLI Mandiri)
Jika tidak menggunakan Docker Compose, Anda bisa langsung menjalankan perintah ini di terminal:
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

---

## 3. Langkah 2: Konfigurasi Scraping di Prometheus

Agar Prometheus menarik metrik dari cAdvisor, Anda harus mendaftarkannya di file konfigurasi Prometheus.

1. Buka file `prometheus.yml`:
   ```bash
   nano ~/monitoring/prometheus.yml
   ```
2. Tambahkan job `cadvisor` di bawah bagian `scrape_configs`:
   ```yaml
   scrape_configs:
     - job_name: 'prometheus'
       static_configs:
         - targets: ['localhost:9090']

     - job_name: 'cadvisor'
       static_configs:
         # Gunakan 'cadvisor:8080' jika Prometheus berada dalam satu docker-compose network
         # Gunakan 'localhost:8080' atau '<IP_VM>:8080' jika berjalan di luar jaringan kontainer
         - targets: ['cadvisor:8080']
   ```
3. Restart Prometheus untuk menerapkan konfigurasi baru:
   ```bash
   docker restart prometheus
   ```

---

## 4. Langkah 3: Pengujian di Prometheus Web UI

Buka antarmuka Prometheus (`http://<IP_VM>:9090`) dan jalankan query berikut pada kolom pencarian:

* **Melihat semua kontainer yang aktif:**
  ```promql
  container_last_seen{name!=""}
  ```
* **Menghitung total kontainer yang sedang berjalan:**
  ```promql
  count(container_last_seen{name!=""})
  ```
* **Menghitung uptime/lama waktu kontainer berjalan (dalam detik):**
  ```promql
  time() - container_start_time_seconds{name!=""}
  ```

---

## 5. Pemecahan Masalah (Troubleshooting)

### A. Error Parsing YAML saat Start Prometheus
* **Masalah**: Prometheus gagal dijalankan (*CrashLoopBackOff* atau status berhenti).
* **Penyebab**: Karakter ilegal seperti `├` atau kesalahan spasi/indentasi pada file `prometheus.yml`.
* **Solusi**: Pastikan sebelum tanda `- targets:` diisi oleh spasi biasa, bukan karakter grafis terminal hasil copy-paste.

### B. cAdvisor Menampilkan Nilai 0 atau Kosong untuk Disk
* **Penyebab**: cAdvisor membutuhkan hak akses istimewa untuk membaca statistik filesystem host.
* **Solusi**: Pastikan flag `--privileged` (atau `privileged: true` pada docker-compose) dan mounting `/var/lib/docker` sudah dikonfigurasi dengan benar.
