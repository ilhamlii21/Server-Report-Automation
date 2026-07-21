# Dokumentasi Workflow: Server Report n8n 1.0

Dokumen ini berisi penjelasan teknis dan panduan workflow `Server Report n8n 1.0` yang digunakan untuk mengotomatisasi pembuatan laporan performa server, pemeriksaan status service, perenderan grafik Grafana, dan pengiriman laporan ke Notion.

---

## 1. Deskripsi Umum

Workflow ini berfungsi untuk:
- Mengambil metrik sistem (CPU, RAM, Disk, Network) dari **Prometheus**.
- Menjalankan perintah sistem lokal (*Command Execution*) untuk mengecek top proses, service Systemd, PM2, dan Docker.
- Merender panel grafik dari **Grafana** menjadi file gambar `.png`.
- Menyusun dan mengirimkan seluruh ringkasan statistik serta gambar grafik ke halaman **Notion**.

---

## 2. Trigger & Parameter Inisialisasi

- **Trigger**: `Manual Trigger` (`When clicking 'Execute workflow'`) atau `Cron Schedule`.
- **Node Initial Setup (`Edit Fields`)**:
  - `server`: `ilhamvm2`
  - `reported_by`: `n8n Server Automation`
  - `pic`: `xxxx`
  - `from`: Dynamic milli-timestamp 2 jam lalu (`$now.minus({ hours: 2 })`)
  - `to`: Dynamic milli-timestamp saat ini (`$now`)
  - `report_date`: Tanggal laporan (`dd LLLL yyyy`)

---

## 3. Komponen Utama & Pengambilan Data

### A. Query Prometheus (Endpoint: `http://<PROMETHEUS_HOST>:9090/api/v1/query`)

1. **CPU Metrics**:
   - **AVG CPU**: `100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[2h])) * 100)`
   - **PEAK CPU**: `100 - (min(irate(node_cpu_seconds_total{mode="idle"}[2h])) * 100)`

2. **Memory & Swap Metrics**:
   - **AVG RAM**: `avg_over_time(((1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100)[2h:1m])`
   - **PEAK RAM**: `max_over_time(((1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100)[2h:1m])`
   - **Swap Used**: `(node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes) / 1024 / 1024 / 1024` (in GB)
   - **Swap Total**: `node_memory_SwapTotal_bytes / 1024 / 1024 / 1024` (in GB)

3. **Disk Metrics**:
   - **Disk Usage (%)**: `(1 - (node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100`
   - **Pertumbuhan Disk (30d)**: `((node_filesystem_size_bytes{mountpoint="/"}-free) - (size_offset_30d - free_offset_30d)) / 1024 / 1024 / 1024`

4. **Network Metrics**:
   - **Inbound (RX)**: `sum(rate(node_network_receive_bytes_total{device!~"lo"}[2h])) * 8 / 1000000` (in Mbps)
   - **Outbound (TX)**: `sum(rate(node_network_transmit_bytes_total{device!~"lo"}[2h])) * 8 / 1000000` (in Mbps)

5. **PM2 Metrics**:
   - **PM2 List Services**: `pm2_up`
   - **PM2 Restarts**: `pm2_restarts`
   - **PM2 Uptime**: `pm2_uptime`

---

### B. Perintah Sistem (CLI Execution)

- **Proses Pengguna RAM Terbesar**:
  ```bash
  ps -eo %mem,comm --sort=-%mem | head -n 6 | tail -n 5 | awk '{print $1"%  " $2}'
  ```
- **Pemeriksaan Systemd & Zombie Process**:
  ```bash
  FAILED_NAMES=$(systemctl list-units --all --state=failed --no-legend | awk '{print $1}' | paste -sd ", " -); [ -z "$FAILED_NAMES" ] && FAILED_NAMES="Tidak Ada"; echo "ACTIVE:$(systemctl list-units --type=service --state=running | grep -c 'loaded') | FAILED:$(systemctl list-units --all --state=failed | grep -c 'failed') (Detail Gagal: $FAILED_NAMES) | ZOMBIE:$(ps -eo stat | grep -c 'Z')"
  ```
- **Pemeriksaan Directory Log Terbesar**:
  ```bash
  du -h --max-depth=1 /var/log 2>/dev/null | sort -hr | head -n 5
  ```

---

### C. Grafana Image Rendering

Metode pengambilan gambar grafik menggunakan perintah `curl` ke service Grafana Renderer lokal:

```bash
curl -H "Authorization: Bearer xxxx" \
  "http://localhost:3000/render/d-solo/rYdddlPWk/grafanavm2?orgId=1&panelId=<PANEL_ID>&width=1000&height=500&from=<FROM>&to=<TO>&tz=Asia%2FMakassar" \
  -o /var/www/html/renders/<PANEL_NAME>.png
```

- **Panel ID 77**: CPU Graph (`/renders/cpu.png`)
- **Panel ID 78**: RAM Graph (`/renders/ram.png`)
- **Panel ID 152**: Disk Graph (`/renders/disk.png`)

---

### D. Update Laporan ke Notion

Koneksi ke API Notion menggunakan metode HTTP Request `PATCH`:
- **URL**: `https://api.notion.com/v1/blocks/<BLOCK_ID>/children`
- **Headers**:
  - `Authorization`: `Bearer xxxx`
  - `Notion-Version`: `2022-06-28`
  - `Content-Type`: `application/json`

**Data yang dituliskan meliputi:**
1. Rata-rata & Puncak CPU/RAM.
2. % Penggunaan Disk, Pertumbuhan Disk, dan Top 5 Folder Log.
3. Rata-rata Network Inbound (RX) / Outbound (TX).
4. Status Service Systemd (Active/Failed/Zombie).
5. Service PM2 (Status Online, Restart Count, Uptime).
6. Status Container Docker.
7. Penyematan URL Gambar Grafik CPU, RAM, dan Disk.

---

## 4. Keamanan & Konfigurasi Kredensial

Semua informasi sensitif telah disamarkan (`xxxx`):

| Elemen Kredensial | Tipe Kredensial | Nilai Asli (Disamarkan) |
| :--- | :--- | :--- |
| **Grafana API Bearer Token** | Service Account Token | `Bearer xxxx` |
| **Notion API Integration Token**| Internal Integration Token | `Bearer xxxx` |
| **Notion Account Credential ID** | n8n Credential Storage | `id: "xxxx"` |

---
*File dokumentasi ini dibuat tanpa mengubah kode atau workflow asli `Server Report n8n 1.0.json`.*
