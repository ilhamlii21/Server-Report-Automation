# Panduan Revisi: Mengganti Node Execute Command Menjadi HTTP Request (Prometheus & Grafana)

Dokumen ini berisi pemetaan terperinci antara command CLI lama (Execute Command) dengan query PromQL (HTTP Request) yang telah disesuaikan dengan file workflow baru **`Server Report Workflow Tanpa Command Execute (1).json`**.

---

## 1. Konfigurasi Umum Node HTTP Request Prometheus

Untuk semua node query Prometheus:
* **Node Type**: `n8n-nodes-base.httpRequest`
* **Method**: `GET`
* **URL**: `http://20.205.17.18:9090/api/v1/query` (atau URL/IP Prometheus lokal/publik Anda)
* **Send Query Parameters**: `true`
* **Query Parameters**:
  - `name`: `query`
  - `value`: (Query PromQL di bawah ini)

---

## 2. Pemetaan Command CLI Lama ke PromQL & Parser JavaScript Baru

Berikut adalah pemetaan setiap bagian monitoring dari script CLI sistem ke query **PromQL** serta cara pemrosesan datanya (Parser):

### A. Top 5 RAM-consuming Processes
* **Command CLI Lama**:
  ```bash
  ps -eo %mem,comm --sort=-%mem | head -n 6 | tail -n 5 | awk '{print $1"%  " $2}'
  ```
* **HTTP Request PromQL Query**:
  ```promql
  topk(5, (sum(namedprocess_namegroup_memory_bytes{memtype="resident"}) by (groupname) / scalar(node_memory_MemTotal_bytes)) * 100)
  ```
* **Penjelasan & Parser**:
  Menggunakan metrics dari `process-exporter` untuk menghitung persentase RAM per grup proses secara langsung. Hasilnya dipetakan di Notion menggunakan ekspresi:
  ```javascript
  const ramProcesses = ($('Prometheus Proses Makan Ram').first()?.json?.data?.result || [])
    .map(r => parseFloat(r.value[1]).toFixed(2) + "%  " + r.metric.groupname)
    .join('\n');
  ```

---

### B. Systemd Services & Zombie Processes
* **Command CLI Lama**:
  ```bash
  FAILED_NAMES=$(systemctl list-units --all --state=failed --no-legend ...); echo "ACTIVE:... | FAILED:... | ZOMBIE:..."
  ```
* **HTTP Request PromQL Query**:
  ```promql
  ({__name__='node_systemd_unit_state', state='active'} == 1) or ({__name__='node_systemd_unit_state', state='failed'} == 1) or {__name__='node_processes_zombies'}
  ```
* **Penjelasan & Parser (JavaScript Code Node)**:
  Query di atas mengambil metrik state systemd (active & failed) serta jumlah zombie process sekaligus. Data ini diproses oleh node **Code in JavaScript** dengan kode berikut:
  ```javascript
  const items = $input.all();
  const results = items[0]?.json?.data?.result || [];

  const activeServices = results.filter(r => r.metric.__name__ === 'node_systemd_unit_state' && r.metric.state === 'active');
  const activeCount = activeServices.length;
  const activeDetails = activeServices.slice(0, 5).map(r => r.metric.name).join(', ') || "Tidak Ada";

  const failedServices = results.filter(r => r.metric.__name__ === 'node_systemd_unit_state' && r.metric.state === 'failed');
  const failedCount = failedServices.length;
  const failedDetails = failedServices.map(r => r.metric.name).join(', ') || "Tidak Ada";

  const zombieMetric = results.find(r => r.metric.__name__ === 'node_processes_zombies');
  const zombieCount = zombieMetric ? parseInt(zombieMetric.value[1]) : 0;

  return {
    stdout: `ACTIVE:${activeCount} (Contoh Aktif: ${activeDetails}) | FAILED:${failedCount} (Detail Gagal: ${failedDetails}) | ZOMBIE:${zombieCount}`
  };
  ```

---

### C. Disk Usage, Pertumbuhan Disk, & Top 5 Folder Terbesar
* **Command CLI Lama**:
  ```bash
  # Disk space & growth, du -h --max-depth=1 /var/log
  du -h --max-depth=1 /var/log 2>/dev/null | sort -hr | head -n 5
  ```
* **HTTP Request PromQL Query**:
  1. **Disk Usage (Root `/`)**:
     ```promql
     (1 - (node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100
     ```
  2. **Pertumbuhan Disk (30 Hari Terakhir)**:
     ```promql
     ((node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"}) - (node_filesystem_size_bytes{mountpoint="/"} offset 30d - node_filesystem_free_bytes{mountpoint="/"} offset 1d)) / 1024 / 1024 / 1024
     ```
  3. **Top 5 Folder Terbesar**:
     ```promql
     topk(5, node_dir_file_size_bytes)
     ```
* **Penjelasan & Parser**:
  Data folder terbesar didapatkan dari custom exporter yang mengirim metrik `node_dir_file_size_bytes`. Data ini dikonversi secara dinamis menggunakan helper bytes formatter di Notion:
  ```javascript
  const top5Result = $('Prometheus Top 5 Folder Terbesar').first()?.json?.data?.result || [];
  function formatBytes(bytes) {
    if (bytes === 0) return '0 B';
    const k = 1024;
    const sizes = ['B', 'KB', 'MB', 'GB', 'TB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return parseFloat((bytes / Math.pow(k, i)).toFixed(1)) + ' ' + sizes[i];
  }
  const top5Text = top5Result.map(item => {
    const path = item.metric.path || 'Unknown';
    const bytes = parseFloat(item.value[1] || 0);
    return `${formatBytes(bytes)}\t${path}`;
  }).join('\n') || "Tidak ada data folder";
  ```

---

### D. Docker Health & Container Stats
* **Command CLI Lama**:
  ```bash
  docker ps --format "{{.Names}} ({{.Status}})" ...
  ```
* **HTTP Request PromQL Query**:
  ```promql
  {__name__=~"node_docker_.*"}
  ```
* **Penjelasan & Parser**:
  Mengambil seluruh metrik Docker Exporter (`node_docker_containers_running`, `node_docker_volumes_count`, `node_docker_images_size_bytes`, `node_docker_images_count`, `node_docker_reclaimable_size_bytes`, `node_docker_container_info`, `node_docker_container_writable_bytes`).
  Parser di Notion membedakan container yang aktif dan menghitung total size image serta space yang bisa dibersihkan (reclaimable space).

---

### E. Cronjob Audit & Cron Error
* **Command CLI Lama**:
  ```bash
  crontab -l ...; journalctl _COMM=cron ...
  ```
* **HTTP Request PromQL Query**:
  ```promql
  {__name__=~"node_cron_.*"}
  ```
* **Penjelasan & Parser**:
  Mengambil metrik `node_cron_job` dan `node_cron_errors_total_24h`.
  Parser di Notion membaca daftar cronjob yang terdaftar (`exported_job`) dan menampilkan status error jika ada kegagalan log dalam 24 jam terakhir.

---

### F. VM Security Audit (SSH & Firewall)
* **Command CLI Lama**:
  ```bash
  SSH_OK=$(journalctl _COMM=sshd ...); FW_STATUS=$(ufw status ...)
  ```
* **HTTP Request PromQL Query**:
  ```promql
  {__name__=~"node_security_.*"}
  ```
* **Penjelasan & Parser**:
  Mengambil metrik `node_security_ssh_success_24h`, `node_security_ssh_fail_24h`, `node_security_suspicious_ip`, `node_security_packages_to_update`, `node_security_ssl_status`, dan `node_security_firewall_status` secara sekaligus untuk ditampilkan lengkap di Notion.

---

### G. Deteksi Lonjakan Log Error
* **Command CLI Lama**:
  ```bash
  ERR_COUNT=$(journalctl -p err..emerg --since "24 hours ago" ...)
  ```
* **HTTP Request PromQL Query**:
  ```promql
  {__name__=~"node_log_.*"}
  ```
* **Penjelasan & Parser**:
  Mengambil metrik `node_log_errors_total_24h` dan `node_log_error_patterns`. Jika total error melebihi batas aman (misal > 50), status akan berubah menjadi Spikes (`Yes`).

---

### H. VM Change Log (30 Hari Terakhir)
* **Command CLI Lama**:
  ```bash
  PKG=$(grep -E "install|upgrade" /var/log/dpkg.log ...); NGINX_CHG=$(find /etc/nginx ...)
  ```
* **HTTP Request PromQL Query**:
  ```promql
  {__name__=~"node_changelog_.*"}
  ```
* **Penjelasan & Parser**:
  Mengambil data audit perubahan sistem meliputi `node_changelog_packages_updated`, `node_changelog_os_version`, `node_changelog_nginx_changes`, `node_changelog_cron_changes`, `node_changelog_firewall_changes`, `node_changelog_new_docker_images`, dan `node_changelog_new_services`.

---

## 3. Optimasi Grafik Panel Grafana (Menggantikan Curl CLI)

Di dalam workflow, node **Execute Command** lama yang menggunakan `curl` untuk mendownload visualisasi grafik panel Grafana telah diganti secara penuh menggunakan node **HTTP Request (n8n)** native dengan response format **`File`**.

### Konfigurasi Baru:
* **Node Type**: `n8n-nodes-base.httpRequest`
* **Method**: `GET`
* **URL**: `http://localhost:3000/render/d-solo/rYdddlPWk/grafanavm2` (atau endpoint Grafana Anda)
* **Headers / Authentication**:
  - `Authorization`: `Bearer <SERVICE_ACCOUNT_TOKEN>`
* **Query Parameters**:
  - `orgId`: `1`
  - `panelId`: (ID Panel yang sesuai, misal: `77`, `78`, `152`)
  - `width`: `1000`
  - `height`: `500`
  - `from`: `{{ $json.from }}`
  - `to`: `{{ $json.to }}`
* **Response Format**: `File`
* **Penjelasan**: Output berupa file binary langsung dikirimkan ke Notion tanpa perlu disimpan ke storage disk lokal terlebih dahulu.
