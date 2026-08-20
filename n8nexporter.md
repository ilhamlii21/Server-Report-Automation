# Panduan Konfigurasi Pelacakan n8n dengan Prometheus (Built-in Metrics)

Dokumen ini menjelaskan langkah-langkah untuk mengaktifkan fitur **built-in metrics** pada **n8n self-hosted** dan mengonfigurasinya dengan **Prometheus** untuk memantau kesehatan n8n serta statistik eksekusi workflow secara otomatis.

---

## 1. Pendahuluan

Secara default, n8n menyediakan endpoint internal `/metrics` yang menyajikan data statistik performa dalam format yang dapat dibaca oleh Prometheus. Fitur ini dinonaktifkan secara bawaan. Dengan mengaktifkannya melalui variabel lingkungan (*environment variables*), kita dapat melacak jumlah eksekusi workflow sukses/gagal, rata-rata durasi eksekusi, serta latensi webhook tanpa memerlukan modul tambahan.

---

## 2. Langkah 1: Mengaktifkan Metrics di n8n (Docker / Docker Compose)

Perbarui konfigurasi container n8n Anda (misalnya di `docker-compose.yml` atau file `.env`) dengan menambahkan variabel lingkungan berikut:

```yaml
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    # ... konfigurasi lainnya ...
    environment:
      - N8N_METRICS=true                               # Mengaktifkan endpoint /metrics
      - N8N_METRICS_INCLUDE_WORKFLOW_INFO=true         # Menyertakan informasi nama/ID workflow pada metrik
      - N8N_METRICS_INCLUDE_WEBHOOK_METRICS=true        # Menyertakan statistik request webhook
      - N8N_METRICS_INCLUDE_QUEUE_METRICS=true          # Menyertakan metrik queue/worker (opsional, jika menggunakan mode antrean)
```

Terapkan perubahan dengan me-restart container n8n Anda:
```bash
docker compose up -d n8n
```

---

## 3. Langkah 2: Konfigurasi Scraping di Prometheus

Agar Prometheus dapat mengambil data dari endpoint metrics n8n, tambahkan konfigurasi job `n8n` di bawah bagian `scrape_configs` pada berkas `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'n8n'
    scrape_interval: 15s
    metrics_path: /metrics
    static_configs:
      # Gunakan nama container 'n8n:5678' jika berada dalam satu docker-compose network.
      # Gunakan '172.17.0.1:5678' atau '<IP_VM>:5678' jika berjalan di luar jaringan kontainer.
      - targets: ['n8n:5678']
```

Restart Prometheus untuk memuat konfigurasi baru:
```bash
docker compose restart prometheus
```

---

## 4. Langkah 3: Verifikasi & Query PromQL yang Tersedia

### **A. Verifikasi Target**
Buka Prometheus Web UI (`http://<IP_VM>:9090/targets`) dan pastikan job **`n8n`** sudah berstatus **`UP`** (hijau).

### **B. Query PromQL Umum**

Berikut adalah query PromQL yang dapat Anda gunakan di Prometheus atau Grafana:

| Kebutuhan | Query |
|---|---|
| **Total eksekusi workflow yang sukses** | `n8n_workflow_success_total` |
| **Total eksekusi workflow yang gagal** | `n8n_workflow_failure_total` |
| **Rata-rata durasi eksekusi workflow (detik)** | `rate(n8n_workflow_execution_duration_seconds_sum[5m]) / rate(n8n_workflow_execution_duration_seconds_count[5m])` |
| **Jumlah workflow yang sedang aktif** | `n8n_workflow_active` |
| **Latensi request webhook** | `n8n_webhook_request_duration_seconds_bucket` |

---

## 5. Pemecahan Masalah (Troubleshooting)

### A. Endpoint `/metrics` Mengembalikan Error 404 atau Terkunci
* **Penyebab**: Variabel lingkungan `N8N_METRICS=true` belum dimuat dengan benar atau kontainer n8n belum direstart.
* **Solusi**: Pastikan variabel tersebut dideklarasikan di bawah bagian `environment:` pada `docker-compose.yml`, jalankan `docker compose down && docker compose up -d` untuk memastikan kontainer membaca konfigurasi baru.

### B. Keamanan Endpoint
* **Peringatan**: Secara bawaan, endpoint `/metrics` tidak memerlukan autentikasi. Pastikan port n8n (`5678`) atau endpoint `/metrics` tidak terbuka secara langsung ke publik. Batasi akses hanya untuk IP internal Prometheus melalui aturan firewall (UFW/Azure Network Security Group).
