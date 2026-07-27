# Panduan Revisi: Mengganti Node Execute Command Menjadi HTTP Request (Prometheus & Grafana)

Dokumen ini berisi instruksi terperinci untuk AI agar dapat memodifikasi workflow n8n (seperti `Server Report n8n 1.0.json` atau workflow lainnya) dengan mengganti semua node **Execute Command (CLI)** menjadi node **HTTP Request** yang langsung memanggil **Prometheus API** atau **Grafana Render API**. 

Asumsi: Server target tidak memerlukan konfigurasi tambahan, sehingga data diambil dari metrik standar Prometheus (`node_exporter`, `process-exporter`, `pm2-exporter`) dan API bawaan Grafana.

---

## 1. Konfigurasi Umum Node HTTP Request Prometheus

Untuk setiap penggantian command CLI sistem ke Prometheus:
* **Node Type**: `n8n-nodes-base.httpRequest`
* **Method**: `GET`
* **URL**: `http://localhost:9090/api/v1/query` (atau URL/IP Prometheus sesuai konfigurasi)
* **Send Query Parameters**: `true`
* **Query Parameters**:
  - `name`: `query`
  - `value`: (Masukkan PromQL Query yang sesuai di bawah ini)

---

## 2. Pemetaan Node CLI Command ke PromQL Query

Berikut adalah pemetaan command CLI yang sebelumnya berjalan di node **Execute Command** ke query **PromQL** pada node **HTTP Request**:

### A. Top 5 RAM-consuming Processes
* **Command CLI Lama**:
  ```bash
  ps -eo %mem,comm --sort=-%mem | head -n 6 | tail -n 5 | awk '{print $1"%  " $2}'
  ```
* **HTTP Request PromQL Query**:
  ```promql
  topk(5, sum(namedprocess_namegroup_memory_bytes{memtype="resident"}) by (groupname))
  ```
  *(Catatan: Menggunakan metrik dari `process-exporter`. Jika process-exporter tidak aktif, gunakan query persentase RAM total sebagai representasi: `(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100`)*

### B. Systemd Services & Zombie Processes
* **Command CLI Lama**:
  ```bash
  FAILED_NAMES=$(systemctl list-units --all --state=failed --no-legend ...); echo "ACTIVE:... | FAILED:... | ZOMBIE:..."
  ```
* **HTTP Request PromQL Query**:
  Untuk mendapatkan status detail unit systemd yang gagal dan zombie process, gunakan query terpisah atau gabungkan di n8n:
  1. **Zombie Processes Count**:
     ```promql
     sum(node_processes_state{state="zombie"})
     ```
  2. **Failed Systemd Services Count**:
     ```promql
     sum(node_systemd_unit_state{state="failed"})
     ```
  3. **Detail Nama Service yang Gagal**:
     ```promql
     node_systemd_unit_state{state="failed"} == 1
     ```

### C. Top 5 Folder Terbesar di /var/log / Disk Space
* **Command CLI Lama**:
  ```bash
  du -h --max-depth=1 /var/log 2>/dev/null | sort -hr | head -n 5
  ```
* **HTTP Request PromQL Query**:
  Karena Prometheus secara default tidak mendeteksi ukuran folder secara dinamis tanpa custom script exporter, data ini digantikan dengan memantau kapasitas partisi disk utama (Root `/`):
  ```promql
  (1 - (node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100
  ```

### D. Docker Health & Container Stats
* **Command CLI Lama**:
  ```bash
  echo "===RUNNING===" && docker ps --format "{{.Names}} ({{.Status}})" ...
  ```
* **HTTP Request PromQL Query**:
  Jika Docker Daemon metrics / cAdvisor diaktifkan pada Prometheus:
  1. **Jumlah Container Berjalan**:
     ```promql
     engine_daemon_container_states_containers{state="running"}
     ```
  2. **Jumlah Container Berhenti / Stopped**:
     ```promql
     engine_daemon_container_states_containers{state="stopped"}
     ```

### E. Cronjob Audit & Cron Error
* **Command CLI Lama**:
  ```bash
  LIST=$(crontab -l ...); FAIL=$(journalctl _COMM=cron ...); echo "FAIL:$FAIL | CHANGE:... ===CRONLIST=== $LIST"
  ```
* **HTTP Request PromQL Query**:
  1. **Status Service Cron Daemon**:
     ```promql
     node_systemd_unit_state{name="cron.service", state="active"}
     ```
  2. **Uptime Cron Service**:
     ```promql
     time() - node_systemd_unit_start_time_seconds{name="cron.service"}
     ```

### F. VM Security Audit (SSH & Firewall)
* **Command CLI Lama**:
  ```bash
  SSH_OK=$(journalctl _COMM=sshd ...); FW_STATUS=$(ufw status ...); echo "SSH:... | UPDATES:... | FW:..."
  ```
* **HTTP Request PromQL Query**:
  1. **Status Service SSH**:
     ```promql
     node_systemd_unit_state{name="ssh.service", state="active"}
     ```
  2. **Status Firewall (UFW) Service**:
     ```promql
     node_systemd_unit_state{name="ufw.service", state="active"}
     ```
  3. **Package Upgrades Pending** (jika `apt` atau `system-update` exporter aktif):
     ```promql
     apt_upgrades_pending
     ```

### G. Deteksi Lonjakan Log Error
* **Command CLI Lama**:
  ```bash
  ERR_COUNT=$(journalctl -p err..emerg --since "24 hours ago" ...); echo "SPIKE:$SPIKE | PATTERNS:$ERR_PATTERNS"
  ```
* **HTTP Request PromQL Query**:
  * Jika menggunakan Loki (terintegrasi dengan Prometheus):
    ```promql
    sum(count_over_time({syslog_identifier=~".+"} |= "error"[24h]))
    ```
  * Jika hanya Prometheus murni (tanpa Loki):
    Gunakan total unit systemd yang gagal sebagai indikator error sistem:
    ```promql
    sum(node_systemd_unit_state{state="failed"})
    ```

### H. VM Change Log (30 Hari Terakhir)
* **Command CLI Lama**:
  ```bash
  PKG=$(grep -E "install|upgrade" /var/log/dpkg.log ...); NGINX_CHG=$(find /etc/nginx ...); echo "PKG:... | NGINX:..."
  ```
* **HTTP Request PromQL Query**:
  Untuk melihat perubahan log/sistem melalui metrik:
  1. **Deteksi VM Reboot (Uptime Reset) 30 Hari Terakhir**:
     ```promql
     changes(node_boot_time_seconds[30d])
     ```
  2. **Perubahan Status Service 30 Hari Terakhir**:
     ```promql
     changes(node_systemd_unit_state{state="failed"}[30d])
     ```

---

## 3. Instruksi Mengganti Node Render Grafana (Curl CLI)

Di dalam workflow terdapat node **Execute Command** yang menjalankan perintah `curl` untuk mendownload grafik Grafana Panel, contoh:
```bash
curl -H "Authorization: Bearer <TOKEN>" "http://localhost:3000/render/d-solo/rYdddlPWk/grafanavm2?orgId=1&panelId=77&width=1000&height=500&from=1784517502013&to=1784524702024" -o /var/www/html/renders/cpu.png
```

### Konfigurasi Baru Menggunakan HTTP Request Node (Bawaan n8n):
* **Node Type**: `n8n-nodes-base.httpRequest`
* **Method**: `GET`
* **URL**: `http://localhost:3000/render/d-solo/rYdddlPWk/grafanavm2` (sesuai URL dashboard Grafana)
* **Authentication**: `Header`
  - Name: `Authorization`
  - Value: `Bearer <SERVICE_ACCOUNT_TOKEN>`
* **Send Query Parameters**: `true`
  - `orgId`: `1`
  - `panelId`: (ID Panel yang sesuai, misal: `77`, `78`, `152`)
  - `width`: `1000`
  - `height`: `500`
  - `from`: `{{ $json.from }}` (ambil secara dinamis dari node tanggal sebelumnya)
  - `to`: `{{ $json.to }}` (ambil secara dinamis dari node tanggal sebelumnya)
* **Response Format**: `File` (Sangat Penting! Ini akan menghasilkan binary data gambar `.png` yang siap dikirim/di-upload tanpa perlu menulis file ke disk server lokal).

---

## 4. Penyesuaian Node Code (JavaScript Parser)

Karena input ke node Code berubah dari **text stdout** (hasil CLI shell command) menjadi **JSON** (hasil respon Prometheus HTTP API), kode parser JavaScript di dalam workflow harus disesuaikan.

### Contoh Penyesuaian Parsing RAM & Processes:
* **Sebelumnya (Membaca stdout):**
  ```javascript
  const stdout = $('Execute Command').first().json.stdout;
  // Parser membagi baris text stdout...
  ```
* **Sesudahnya (Membaca JSON dari HTTP Request Prometheus):**
  ```javascript
  const prometheusData = $('HTTP Request Prometheus RAM').first().json.data.result;
  
  // Format hasil dari Prometheus:
  let topProcesses = [];
  for (const item of prometheusData) {
    const groupName = item.metric.groupname || 'Unknown';
    const ramBytes = parseFloat(item.value[1]);
    const ramMB = (ramBytes / 1024 / 1024).toFixed(2);
    topProcesses.push(`${ramMB} MB - ${groupName}`);
  }
  
  return [{
    json: {
      top_processes: topProcesses.join('\n')
    }
  }];
  ```

---

## 5. Panduan Langkah Demi Langkah bagi AI untuk Melakukan Edit

Jika AI ditugaskan untuk mengedit file JSON workflow secara langsung:
1. **Analisis File JSON**: Cari semua node dengan `"type": "n8n-nodes-base.executeCommand"`.
2. **Identifikasi Kegunaan**: Baca parameter `"command"` dari node tersebut untuk mengidentifikasi apakah itu node query system (CPU, RAM, dsb) atau node render Grafana (`curl`).
3. **Ganti Tipe Node**: Ubah tipe node menjadi `"n8n-nodes-base.httpRequest"`.
4. **Isi Parameter HTTP Request**:
   - Gunakan PromQL Query yang sesuai untuk menggantikan logika CLI.
   - Atur parameters URL, method, headers, dan query params sesuai panduan di atas.
5. **Update Koneksi & Node Berikutnya**:
   - Hubungkan output HTTP Request ke node parser/Code.
   - Perbarui kode JavaScript di node Code agar membaca data dari path `.json.data.result` Prometheus, bukan `.json.stdout`.
6. **Verifikasi Sintaksis JSON**: Pastikan file JSON workflow tetap valid setelah diedit.
