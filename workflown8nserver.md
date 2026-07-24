# Dokumentasi Alur Kerja n8n: Server Report Workflow (workflown8nserver.md)

Dokumen ringkas mengenai query Prometheus dan command CLI yang digunakan dalam workflow n8n untuk Laporan Bulanan Server.

---

## 1. Query Prometheus (HTTP Request)

Query berikut dikirimkan ke Prometheus API (`http://<PROMETHEUS_IP>:<PORT>/api/v1/query`) untuk mendapatkan metrik performa.

### CPU
* **AVG CPU Usage (2 Jam terakhir)**:
```text
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[2h])) * 100)
```
* **PEAK CPU Usage (2 Jam terakhir)**:
```text
100 - (min(irate(node_cpu_seconds_total{mode="idle"}[2h])) * 100)
```

### Memory & Swap
* **AVG RAM Usage**:
```text
avg_over_time(((1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100)[2h:1m])
```
* **PEAK RAM Usage**:
```text
max_over_time(((1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100)[2h:1m])
```
* **Swap Used (GB)**:
```text
(node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes) / 1024 / 1024 / 1024
```
* **Swap Total (GB)**:
```text
node_memory_SwapTotal_bytes / 1024 / 1024 / 1024
```

### Disk
* **Disk Usage Partisi Root (%)**:
```text
(1 - (node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100
```
* **Pertumbuhan Disk dibanding 30 Hari Lalu (GB)**:
```text
((node_filesystem_size_bytes{mountpoint="/"}-node_filesystem_free_bytes{mountpoint="/"}) - (node_filesystem_size_bytes{mountpoint="/"}-node_filesystem_free_bytes{mountpoint="/"}) offset 30d) / 1024 / 1024 / 1024
```

### Network Bandwidth
* **Network Inbound RX (Mbps)**:
```text
sum(rate(node_network_receive_bytes_total{device!~"lo"}[2h])) * 8 / 1000000
```
* **Network Outbound TX (Mbps)**:
```text
sum(rate(node_network_transmit_bytes_total{device!~"lo"}[2h])) * 8 / 1000000
```

### PM2 (Node.js)
* **PM2 Services List**:
```text
pm2_up
```
* **PM2 Restarts Count**:
```text
pm2_restarts
```
* **PM2 Uptime**:
```text
pm2_uptime
```

---

## 2. Command CLI Server (Execute Command)

Perintah shell berikut dieksekusi langsung di server lokal untuk audit kesehatan dan status VM.

* **Top 5 Proses Makan RAM**:
```bash
ps -eo %mem,comm --sort=-%mem | head -n 6 | tail -n 5 | awk '{print $1"%  " $2}'
```

* **Top 5 Folder Terbesar di /var/log**:
```bash
du -h --max-depth=1 /var/log 2>/dev/null | sort -hr | head -n 5
```

* **Status Service Systemd & Zombie Process**:
```bash
FAILED_NAMES=$(systemctl list-units --all --state=failed --no-legend | awk '{print $1}' | paste -sd ", " -); [ -z "$FAILED_NAMES" ] && FAILED_NAMES="Tidak Ada"; echo "ACTIVE:$(systemctl list-units --type=service --state=running | grep -c 'loaded') | FAILED:$(systemctl list-units --all --state=failed | grep -c 'failed') (Detail Gagal: $FAILED_NAMES) | ZOMBIE:$(ps -eo stat | grep -c 'Z')"
```

* **Docker Health & Container Stats**:
```bash
RUNNING=$(docker ps --format "{{.Names}} ({{.Status}})" | paste -sd "\n" -); [ -z "$RUNNING" ] && RUNNING="Tidak ada container running"; VOL_SIZE=$(docker system df | grep "Local Volumes" | awk '{print $3}') ; IMG_SIZE=$(docker system df | grep "Images" | awk '{print $4" images ("$3")"}') ; RESTART=$(docker inspect --format '{{.Name}}: {{.RestartCount}}x restart' $(docker ps -q) 2>/dev/null | paste -sd ", " -); [ -z "$RESTART" ] && RESTART="Normal"; CLEANUP=$(docker system df | grep "Images" | awk '{print $5" Reclaimable"}') ; echo "VOL:$VOL_SIZE | IMG:$IMG_SIZE | RESTART:$RESTART | CLEANUP:$CLEANUP ===CONTAINERS=== $RUNNING"
```

* **Daftar Cronjob & Log Error Cron**:
```bash
LIST=$(crontab -l 2>/dev/null | grep -v '^#' | grep -v '^$'); [ -z "$LIST" ] && LIST="Tidak ada cronjob aktif"; FAIL=$(journalctl _COMM=cron --since "24 hours ago" 2>/dev/null | grep -iE "error|fail|failed" | wc -l); echo "FAIL:$FAIL error | CHANGE:No ===CRONLIST=== $LIST"
```

* **Audit Keamanan VM (SSH & Firewall)**:
```bash
SSH_OK=$(journalctl _COMM=sshd --since "24 hours ago" 2>/dev/null | grep -i "Accepted" | wc -l); SSH_FAIL=$(journalctl _COMM=sshd --since "24 hours ago" 2>/dev/null | grep -i "Failed" | wc -l); SUSP_IP=$(journalctl _COMM=sshd --since "24 hours ago" 2>/dev/null | grep -i "Failed" | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head -n 1 | awk '{print $2}'); UPDATES=$(apt list --upgradable 2>/dev/null | grep -c 'upgradable'); FW_STATUS=$(ufw status 2>/dev/null | head -n 1 | awk '{print $2}'); echo "SSH:$SSH_OK OK / $SSH_FAIL Fail | SUSPIP:$SUSP_IP | UPDATES:$UPDATES | SSL:Active | FW:$FW_STATUS"
```

* **Deteksi Lonjakan Log Error**:
```bash
ERR_COUNT=$(journalctl -p err..emerg --since "24 hours ago" 2>/dev/null | wc -l); ERR_PATTERNS=$(journalctl -p err..emerg --since "24 hours ago" 2>/dev/null | awk '{print $5}' | sort | uniq -c | sort -nr | head -n 3 | paste -sd ", " -); echo "SPIKE:$ERR_COUNT | PATTERNS:$ERR_PATTERNS"
```

* **VM Change Log (30 Hari Terakhir)**:
```bash
PKG=$(grep -E "install|upgrade" /var/log/dpkg.log 2>/dev/null | wc -l); NGINX_CHG=$(find /etc/nginx -type f -mtime -30 2>/dev/null | wc -l); CRON_CHG=$(find /etc/cron* /var/spool/cron -type f -mtime -30 2>/dev/null | wc -l); DOCKER_NEW=$(docker images --format "{{.Repository}}" 2>/dev/null | head -n 2); SVC_NEW=$(find /etc/systemd/system -type f -mtime -30 2>/dev/null | wc -l); echo "PKG:$PKG | NGINX:$NGINX_CHG | CRON:$CRON_CHG | DOCKER:$DOCKER_NEW | SVC:$SVC_NEW"
```

---

## 3. Render Dashboard Grafana (curl)

* **Render CPU grafik**:
```bash
curl -H "Authorization: Bearer <GRAFANA_RENDER_TOKEN>" "http://localhost:3000/render/d-solo/rYdddlPWk/grafanavm2?orgId=1&panelId=77&width=1000&height=500&from=1784517502013&to=1784524702024" -o /var/www/html/renders/cpu.png
```
* **Render RAM grafik**:
```bash
curl -H "Authorization: Bearer <GRAFANA_RENDER_TOKEN>" "http://localhost:3000/render/d-solo/rYdddlPWk/grafanavm2?orgId=1&panelId=78&width=1000&height=500&from=1784517502013&to=1784524702024" -o /var/www/html/renders/ram.png
```
* **Render Disk grafik**:
```bash
curl -H "Authorization: Bearer <GRAFANA_RENDER_TOKEN>" "http://localhost:3000/render/d-solo/rYdddlPWk/grafanavm2?orgId=1&panelId=152&width=1000&height=500&from=1784517502013&to=1784524702024" -o /var/www/html/renders/disk.png
```
