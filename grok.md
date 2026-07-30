# Grok Exporter — Dokumentasi Instalasi

Dokumentasi ini mencatat seluruh langkah instalasi **grok_exporter** untuk menangkap metrik SSH login (success/fail) dari `/var/log/auth.log` dan mengeksposnya ke Prometheus.

---

## Arsitektur

```
/var/log/auth.log → grok_exporter → Prometheus → n8n → Notion
```

- **grok_exporter** membaca file log dan mencocokkan pola regex
- Hasil diekspos sebagai metrik Prometheus di port `9144`
- Prometheus men-scrape metrik tersebut setiap interval tertentu

---

## Prasyarat

- Docker & Docker Compose sudah terinstall
- Prometheus sudah berjalan di jaringan `monitoring`
- File `/var/log/auth.log` ada di server (Ubuntu/Debian)

---

## Step 1: Buat File Konfigurasi `grok_config.yml`

Buat file konfigurasi di folder monitoring:

```bash
cat > ~/monitoring/grok_config.yml << 'CONFIGEOF'
global:
  config_version: 3

input:
  type: file
  paths:
    - /var/log/auth.log
  readall: false

metrics:
  - type: counter
    name: ssh_login_success_total
    help: Total SSH login berhasil
    match: 'Accepted (?<method>\S+) for (?<user>\S+) from (?<client_ip>\S+)'
    labels:
      client_ip: '{{.client_ip}}'

  - type: counter
    name: ssh_login_fail_total
    help: Total SSH login gagal
    match: 'Failed (?<method>\S+) for .* from (?<client_ip>\S+)'
    labels:
      client_ip: '{{.client_ip}}'

server:
  port: 9144
CONFIGEOF
```

> **Catatan:** Gunakan sintaks oniguruma `(?<name>...)` bukan `(?P<name>...)` karena grok_exporter menggunakan library oniguruma, bukan Go regex standar.

---

## Step 2: Buat `Dockerfile.grok`

Image resmi `ghcr.io/fstab/grok_exporter` tidak dapat diakses dari server, sehingga kita build dari source code:

```bash
cat > ~/monitoring/Dockerfile.grok << 'EOF'
FROM golang:1.21-alpine AS builder
RUN apk add --no-cache git gcc musl-dev oniguruma-dev
RUN git clone https://github.com/fstab/grok_exporter.git /build
WORKDIR /build
RUN CGO_ENABLED=1 go build -o grok_exporter .

FROM alpine:latest
RUN apk add --no-cache oniguruma
COPY --from=builder /build/grok_exporter /usr/local/bin/grok_exporter
EXPOSE 9144
ENTRYPOINT ["grok_exporter", "-config", "/etc/grok_exporter/config.yml"]
EOF
```

> **Kenapa build dari source?**
> - `ghcr.io/fstab/grok_exporter:latest` — access denied dari registry
> - `prom/grok-exporter:latest` — image tidak ada di Docker Hub
> - Binary GitHub releases v0.2.8 — URL 404 tidak ditemukan
> - Solusi: build langsung dari source menggunakan Go multi-stage build

---

## Step 3: Tambahkan ke `docker-compose.yml`

Tambahkan blok berikut ke dalam bagian `services:` di `docker-compose.yml`, **sebelum** root `networks:`:

```yaml
  grok-exporter:
    build:
      context: .
      dockerfile: Dockerfile.grok
    container_name: grok-exporter
    networks:
      default: null
    ports:
      - mode: ingress
        target: 9144
        published: "9144"
        protocol: tcp
    restart: unless-stopped
    volumes:
      - type: bind
        source: /var/log/auth.log
        target: /var/log/auth.log
        read_only: true
        bind: {}
      - type: bind
        source: /home/ilhamvm2/monitoring/grok_config.yml
        target: /etc/grok_exporter/config.yml
        read_only: true
        bind: {}
```

Validasi konfigurasi:

```bash
docker compose config
```

---

## Step 4: Build dan Jalankan

```bash
cd ~/monitoring

# Build image (proses sekitar 2-5 menit karena compile Go)
docker compose build grok-exporter

# Jalankan container
docker compose up -d grok-exporter

# Verifikasi berjalan
docker ps | grep grok

# Cek logs
docker logs grok-exporter --tail 10
```

---

## Step 5: Verifikasi Metrics

```bash
# Test endpoint metrics
curl http://localhost:9144/metrics | grep ssh_login
```

Output yang diharapkan:

```
ssh_login_fail_total{client_ip="1.2.3.4"} 412
ssh_login_fail_total{client_ip="5.6.7.8"} 201
ssh_login_success_total{client_ip="..."} 7
```

---

## Step 6: Daftarkan ke Prometheus

Edit `prometheus.yml`:

```bash
nano ~/monitoring/prometheus.yml
```

Tambahkan job baru di bawah `scrape_configs`:

```yaml
  - job_name: 'grok-exporter'
    static_configs:
      - targets: ['grok-exporter:9144']
```

Restart Prometheus:

```bash
docker restart prometheus
```

---

## Query PromQL yang Tersedia

| Kebutuhan | Query |
|---|---|
| Total login gagal semua IP | `sum(ssh_login_fail_total)` |
| Total login berhasil | `sum(ssh_login_success_total)` |
| IP dengan gagal terbanyak (Suspicious IP) | `topk(1, sum by (client_ip) (ssh_login_fail_total))` |
| Top 5 IP mencurigakan | `topk(5, sum by (client_ip) (ssh_login_fail_total))` |

---

## Troubleshooting

### Error: `Pattern %{WORD:method} not defined`

Grok_exporter v3 tidak memuat built-in pattern secara otomatis.

**Solusi:** Ganti pattern Grok dengan raw regex menggunakan sintaks oniguruma.

---

### Error: `undefined group option`

Sintaks `(?P<name>...)` adalah Python regex, bukan oniguruma.

**Solusi:** Gunakan `(?<name>...)` tanpa huruf P.

```
# Salah
match: 'Accepted (?P<method>\S+)'

# Benar
match: 'Accepted (?<method>\S+)'
```

---

### Error: Image `ghcr.io/fstab/grok_exporter` denied

Registry GitHub Container Registry memerlukan autentikasi atau paket bersifat private.

**Solusi:** Build dari source menggunakan `Dockerfile.grok` seperti di Step 2.

---

### Metrics bernilai 0

Disebabkan `readall: false` yang hanya membaca log baru sejak container start.

**Solusi sementara** untuk melihat data historis:

```bash
sed -i 's/readall: false/readall: true/' ~/monitoring/grok_config.yml
docker stop grok-exporter && docker rm grok-exporter && docker compose up -d grok-exporter
```

Setelah data terbaca, kembalikan ke `false`:

```bash
sed -i 's/readall: true/readall: false/' ~/monitoring/grok_config.yml
docker stop grok-exporter && docker rm grok-exporter && docker compose up -d grok-exporter
```

---

### Container tidak membaca config baru setelah update

`docker restart` terkadang tidak cukup untuk memuat ulang perubahan bind mount.

**Solusi:**

```bash
docker stop grok-exporter
docker rm grok-exporter
docker compose up -d grok-exporter
```

---

### YAML docker-compose.yml Error: `did not find expected key`

Penyebab umum: indentasi tidak konsisten antara services.

**Cek indentasi:**

```bash
cat -An ~/monitoring/docker-compose.yml | grep -n "" | head -30
```

Semua service harus berada di indentasi yang sama di bawah `services:`.
