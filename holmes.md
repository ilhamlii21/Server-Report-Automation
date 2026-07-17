# Panduan Instalasi dan Konfigurasi HolmesGPT

Dokumen ini berisi panduan lengkap untuk memasang (install), mengonfigurasi, dan menguji HolmesGPT di lingkungan VM/Server.

---

## 1. Instalasi HolmesGPT

HolmesGPT dipasang menggunakan manajer paket Python (`pip`). Jalankan perintah berikut di terminal server:

```bash
pip install holmesgpt --break-system-packages
```

---

## 2. Konfigurasi Environment Variables (`~/.bashrc`)

Agar HolmesGPT dapat menggunakan model LLM secara otomatis setiap kali Anda membuka terminal baru, masukkan konfigurasi environment variables berikut ke dalam file `~/.bashrc`.

1. Buka file `.bashrc` dengan editor teks:
   ```bash
   nano ~/.bashrc
   ```

2. Tambahkan baris-baris berikut di bagian paling bawah file:
   ```bash
   # Konfigurasi API Key Gemini
   export GEMINI_API_KEY="AIzaSy..." # Ganti dengan API Key Gemini Anda yang utuh

   # Set Model Standar HolmesGPT
   export MODEL="gemini/gemini-3.5-flash"
   export TOOL_SCHEMA_NO_PARAM_OBJECT_IF_NO_PARAMS=true
   ```

3. Simpan perubahan (`Ctrl + O`, `Enter`, lalu `Ctrl + X`).

4. Muat ulang konfigurasi terminal agar langsung aktif:
   ```bash
   source ~/.bashrc
   ```

---

## 3. Konfigurasi Koneksi Toolset Grafana & Prometheus

Buat file konfigurasi Holmes agar dapat terhubung dengan instansi Grafana dan Prometheus lokal.

1. Pastikan folder konfigurasi `.holmes` sudah dibuat:
   ```bash
   mkdir -p ~/.holmes
   ```

2. Buat dan edit file `config.yaml`:
   ```bash
   nano ~/.holmes/config.yaml
   ```

3. Tempelkan (paste) konfigurasi berikut:
   ```yaml
   toolsets:
     prometheus/metrics:
       enabled: true
       config:
         prometheus_url: "http://localhost:9090"

     grafana/dashboards:
       enabled: true
       config:
         api_url: "http://localhost:3000"
         api_key: "glsa_DCyLlODJ9kngUmdfZHgj9qRjbYFhhjgG_34faf921" # Sesuaikan dengan Token Service Account Grafana Anda
         enable_rendering: true
   ```

4. Simpan file (`Ctrl + O`, `Enter`, lalu `Ctrl + X`).

---

## 4. Cara Penggunaan & Pengujian

### A. Sesi Interaktif (Interactive Mode)
Untuk masuk ke mode chat langsung dengan HolmesGPT, jalankan perintah:
```bash
holmes
```
Di dalam mode ini, Anda dapat langsung mengetik pertanyaan di prompt `User:`. Untuk keluar, gunakan perintah `exit` atau tekan `Ctrl + C`.

### B. Perintah Langsung (Single Ask Mode)
Untuk meminta HolmesGPT menganalisis data secara langsung dari terminal (sangat berguna untuk dipanggil oleh n8n nantinya):
```bash
holmes ask "Analisis panel CPU Basic dari dashboard grafanavm2 selama 30 hari terakhir, apakah ada anomali?"
```
