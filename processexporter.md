# Panduan Instalasi dan Konfigurasi Process Exporter (Docker)

Dokumen ini menjelaskan cara memasang dan mengonfigurasi **Process Exporter** menggunakan Docker untuk memantau penggunaan memori RAM secara granular berdasarkan **User Linux** dan **Aplikasi** yang berjalan di server target.

---

## 1. Pendahuluan

Secara default, Node Exporter hanya mencatat penggunaan resource server secara total. Untuk mendapatkan visibilitas yang lebih dalam (siapa user-nya dan aplikasi apa yang memakan RAM terbesar), kita menggunakan `process-exporter` yang dikonfigurasi untuk mengagregasikan metrik berdasarkan kombinasi username dan command.

---

## 2. Langkah 1: Membuat File Konfigurasi di VM Host

Buat folder dan file konfigurasi di host VM Anda untuk mendefinisikan aturan pengelompokkan metrik:

1. Buat direktori konfigurasi:
   ```bash
   sudo mkdir -p /opt/process-exporter
   ```

2. Buat file `process-exporter.yml`:
   ```bash
   sudo nano /opt/process-exporter/process-exporter.yml
   ```

3. Masukkan konfigurasi berikut:
   ```yaml
   process_names:
     - name: "{{.Username}}:{{.Comm}}"
       cmdline:
         - '.+'
   ```
   *Catatan: Konfigurasi di atas akan mengelompokkan metrik dengan format `User:Aplikasi` (contoh: `ilhamvm2:node`, `root:mysqld`).*

---

## 3. Langkah 2: Menjalankan Container Docker

Jalankan container Docker dengan melakukan mounting ke `/proc` host (untuk mengambil metrik proses) dan `/etc/passwd` host (agar Docker dapat menerjemahkan ID User numerik / UID menjadi nama asli seperti `ilhamvm2`).

```bash
# 1. Hapus container lama jika ada konflik
docker stop process-exporter && docker rm process-exporter

# 2. Jalankan container baru
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

---

## 4. Langkah 3: Integrasi dengan Prometheus

1. Tambahkan target `process-exporter` ke dalam file konfigurasi `prometheus.yml` Anda:
   ```yaml
     - job_name: 'process-exporter-users'
       static_configs:
         - targets:
             - '172.17.0.1:9256'
   ```
   *Catatan: IP `172.17.0.1` adalah IP default gateway dari Docker untuk menghubungi host VM.*

2. Terapkan perubahan dengan me-restart container Prometheus Anda:
   ```bash
   docker compose restart prometheus
   ```

---

## 5. Langkah 4: Visualisasi di Grafana

Buat panel baru di dashboard Grafana Anda dengan ketentuan sebagai berikut:

* **Query PromQL:**
  ```promql
  sum(namedprocess_namegroup_memory_bytes{job="process-exporter-users", memtype="resident"}) by (groupname)
  ```
* **Legend (Legenda):**
  Isi dengan **`{{groupname}}`** untuk menampilkan label kombinasi `User:Aplikasi`.
* **Unit Options (Satuan Y-Axis):**
  Ubah menjadi **Data** -> **Bytes (IEC)** agar nilainya ditampilkan rapi dalam satuan KiB/MiB/GiB.

---

## 6. Langkah 5: Simulasi Pengujian Penggunaan RAM

Untuk memverifikasi apakah alur pendeteksian RAM per User ini berfungsi dengan baik, Anda dapat menjalankan perintah Python berikut untuk meminjam RAM sebesar 200 MB selama 60 detik di terminal VM Anda:

```bash
python3 -c "import time; x = bytearray(200 * 1024 * 1024); time.sleep(60)" &
```

Buka atau refresh panel Grafana Anda dalam rentang waktu 60 detik tersebut (atur range waktu ke **`Last 5 minutes`**). Grafik akan langsung mendeteksi alokasi memori RAM tersebut dengan label **`ilhamvm2:python3`**.
