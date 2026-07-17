# Panduan Instalasi dan Manfaat PM2 Prometheus Exporter

Dokumen ini menjelaskan cara memasang **PM2 Prometheus Exporter** pada VM target dan menjelaskan manfaatnya untuk memantau kesehatan aplikasi Node.js (seperti n8n self-hosted) yang berjalan di bawah PM2.

---

## 1. Pendahuluan

Secara bawaan (*default*), process manager PM2 tidak membagikan data metrik internalnya ke jaringan luar. Agar Prometheus dapat memantau status aplikasi Node.js secara otomatis tanpa menggunakan AI, kita menginstal modul eksportir resmi/pihak ketiga bernama `pm2-prometheus-exporter`. Modul ini membuka port **`9209`** pada VM untuk menyajikan data statistik PM2.

---

## 2. Manfaat Menggunakan PM2 Exporter

Menggunakan eksportir ini memberikan beberapa manfaat utama bagi kestabilan aplikasi:
1. **Pemantauan Status Real-time**: Mengetahui status aplikasi secara instan (`online`, `stopped`, atau `errored`).
2. **Deteksi Crash Otomatis**: Melacak berapa kali aplikasi Anda mengalami crash dan restart melalui metrik akumulasi (`pm2_restarts`). Jika angka restart tinggi, ini mengindikasikan adanya bug atau kehabisan memori (*Out of Memory*).
3. **Pelacakan Durasi Berjalan (Uptime)**: Menghitung secara tepat berapa lama aplikasi telah berjalan sejak dinyalakan terakhir kali.
4. **Pemantauan Resource Akurat**: Melacak konsumsi memori RAM (`pm2_memory`) dan CPU (`pm2_cpu`) khusus untuk setiap aplikasi Node.js individual yang dikelola PM2.
5. **Sistem Peringatan Dini (Alerting)**: Memungkinkan kita membuat alarm di Grafana/Prometheus yang otomatis mengirim pesan jika n8n atau aplikasi penting lainnya mati.

---

## 3. Langkah-Langkah Instalasi & Konfigurasi

### **Langkah 1: Instal Modul Exporter di PM2 (VM Host)**
Jalankan perintah berikut di terminal VM Anda:
```bash
pm2 install pm2-prometheus-exporter
```
*Modul ini akan otomatis terunduh, aktif di PM2, dan mulai mempublikasikan metrik pada port `9209`.*

---

### **Langkah 2: Konfigurasi `prometheus.yml`**
Buka file konfigurasi Prometheus Anda:
```bash
nano ~/monitoring/prometheus.yml
```

Tambahkan blok konfigurasi baru untuk PM2 di bawah bagian `scrape_configs` (sertakan `fallback_scrape_protocol` untuk mengatasi masalah header kosong/blank Content-Type pada Prometheus versi baru):

```yaml
  - job_name: 'pm2-exporter'
    fallback_scrape_protocol: PrometheusText0.0.4
    static_configs:
      - targets:
          - '172.17.0.1:9209'
```
*Catatan: IP `172.17.0.1` adalah IP default gateway dari Docker menuju host VM.*

---

### **Langkah 3: Terapkan Perubahan & Restart Prometheus**
Jalankan perintah ini di dalam direktori `~/monitoring`:
```bash
docker compose restart prometheus
```

---

## 4. Langkah Verifikasi

1. **Cek Status Target:**
   Buka browser dan akses halaman target Prometheus Anda:
   👉 `http://<IP_VM>:9090/targets`
   Pastikan target **`pm2-exporter`** sudah berstatus hijau (**`UP`**).

2. **Cek Penggunaan Metrik di Prometheus:**
   Akses `http://<IP_VM>:9090` dan uji query PromQL berikut:
   * **Mengecek jumlah restart:** `pm2_restarts`
   * **Mengecek status online/offline:** `pm2_status`
   * **Menghitung uptime aplikasi (dalam detik):** `time() - (pm2_uptime / 1000)`
