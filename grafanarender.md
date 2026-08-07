# Panduan Lengkap Instalasi & Konfigurasi Grafana Image Renderer via Docker Compose

Dokumen ini berisi panduan teknis langkah demi langkah untuk menginstal, mengonfigurasi, dan memverifikasi **Grafana Image Renderer** menggunakan **Docker Compose**.

---

## 1. Prasyarat & Konsep Dasar

Grafana Image Renderer adalah layanan tambahan (service) yang menggunakan headless browser (Chromium) untuk merender panel grafik dari Grafana menjadi file gambar format PNG.

- **Grafana Service**: Berperan sebagai antarmuka utama dan penyedia data grafik.
- **Grafana Renderer Service**: Berperan sebagai perender gambar yang menerima permintaan dari Grafana, membuka panel grafik secara headless, dan mengembalikan file gambar `.png`.

---

## 2. Konfigurasi `docker-compose.yml`

Berikut adalah file `docker-compose.yml` lengkap yang sudah disesuaikan agar service Grafana dan Grafana Renderer dapat saling berkomunikasi secara aman dan stabil.

```yaml
version: '3.8'

services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      # Konfigurasi Layanan Rendering
      - GF_RENDERING_SERVER_URL=http://grafana-renderer:8081/render
      - GF_RENDERING_CALLBACK_URL=http://grafana:3000/
      - GF_RENDERING_RENDERER_TOKEN=secret-renderer-token
      - GF_LOG_FILTERS=rendering:debug
    networks:
      - monitoring

  grafana-renderer:
    image: grafana/grafana-image-renderer:latest
    container_name: grafana-renderer
    restart: unless-stopped
    ports:
      - "8081:8081"
    environment:
      - ENABLE_METRICS=true
      - HTTP_HOST=0.0.0.0
      - AUTH_TOKEN=secret-renderer-token
      - BROWSER_TZ=Asia/Jakarta
    networks:
      - monitoring

volumes:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

---

## 3. Penjelasan Variabel Lingkungan (Environment Variables)

### Service `grafana`
| Variabel | Fungsi |
| :--- | :--- |
| `GF_RENDERING_SERVER_URL` | URL internal tempat Grafana mengirim request rendering (misal: `http://grafana-renderer:8081/render`). |
| `GF_RENDERING_CALLBACK_URL` | URL internal tempat kontainer renderer mengakses Grafana untuk mengambil tampilan halaman dashboard (misal: `http://grafana:3000/`). |
| `GF_RENDERING_RENDERER_TOKEN` | Token keamanan internal. **Harus bernilai sama** dengan `AUTH_TOKEN` pada service `grafana-renderer`. |
| `GF_LOG_FILTERS` | Mengaktifkan log detail (debug) khusus untuk proses rendering. |

### Service `grafana-renderer`
| Variabel | Fungsi |
| :--- | :--- |
| `ENABLE_METRICS` | Mengaktifkan metrik performa renderer. |
| `HTTP_HOST` | Mengatur IP listening kontainer (dibuat `0.0.0.0` agar dapat diakses dari kontainer lain). |
| `AUTH_TOKEN` | Token autentikasi internal renderer. **Harus bernilai sama** dengan `GF_RENDERING_RENDERER_TOKEN` milik Grafana. |
| `BROWSER_TZ` | Mengatur zona waktu browser headless (misal: `Asia/Jakarta` atau `Asia/Makassar`) agar waktu grafik pada gambar sesuai. |

> ⚠️ **Catatan Penting Docker Network:**
> Kedua service (`grafana` dan `grafana-renderer`) **WAJIB** berada di dalam Docker network yang sama (contoh: `monitoring`) agar resolusi nama domain/hostname internal `grafana-renderer` dapat dikenali oleh Grafana.

---

## 4. Langkah-Langkah Instalasi & Pengoperasian

1. **Buat / Edit File `docker-compose.yml`**
   Buka terminal VM dan salin konfigurasi di atas ke file `docker-compose.yml`.

2. **Jalankan Service**
   Jalankan perintah berikut untuk mengunduh image dan menyalakan kontainer:
   ```bash
   docker compose up -d
   ```

3. **Verifikasi Status Kontainer**
   Pastikan kedua kontainer dalam status `Up` / `healthy`:
   ```bash
   docker compose ps
   ```

4. **Periksa Log Sistem**
   Untuk memastikan tidak ada eror saat inisialisasi:
   ```bash
   # Log Grafana
   docker logs grafana

   # Log Grafana Renderer
   docker logs grafana-renderer
   ```

---

## 5. Panduan Pengujian (Testing & Verification)

### A. Pengujian via UI Grafana
1. Buka Grafana di browser (misal: `http://<IP-VM>:3000`).
2. Pilih salah satu **Dashboard** dan buka salah satu **Panel Grafik**.
3. Klik dropdown judul panel -> **Share**.
4. Pilih tab **Link**, lalu klik tombol **Generate image** atau **Direct link rendered image**.
5. Grafik akan dirender menjadi gambar `.png` yang dapat didownload/dibuka.

### B. Pengujian via Terminal / `curl` (API)
Untuk menguji pengambilan gambar grafik secara otomatis via skrip atau n8n:

```bash
curl -s -u admin:admin \
  "http://localhost:3000/render/d-solo/<DASHBOARD_UID>/<SLUG>?orgId=1&panelId=<PANEL_ID>&width=1000&height=500" \
  -o /tmp/hasil_render.png
```

Cek file hasil render:
```bash
ls -lh /tmp/hasil_render.png
```
*Jika ukuran file lebih dari 10KB (bukan 0 Byte dan bukan pesan error HTML), maka perenderan berhasil.*

---

## 6. Troubleshooting & Solusi Masalah Umum

### 1. Error: `Using the default [rendering]renderer_token is not allowed...`
- **Penyebab:** Pada Grafana versi 11+, nama variabel token di sisi Grafana harus menggunakan `GF_RENDERING_RENDERER_TOKEN` (bukan `GF_RENDERING_AUTH_TOKEN`).
- **Solusi:** Ganti nama variabel lingkungan di service `grafana` menjadi `GF_RENDERING_RENDERER_TOKEN=secret-renderer-token`.

### 2. Error: `dial tcp: lookup grafana-renderer on 127.0.0.11:53: server misbehaving`
- **Penyebab:** Service `grafana` dan `grafana-renderer` berada pada Docker network yang berbeda.
- **Solusi:** Pastikan blok `networks:` dengan nama network yang sama tercantum di bawah **kedua** service pada `docker-compose.yml`.

### 3. Error: `Failed to generate image` pada UI
- **Penyebab:** Kontainer `grafana-renderer` tidak dapat mengakses callback URL Grafana, atau request timeout.
- **Solusi:** 
  1. Cek `GF_RENDERING_CALLBACK_URL=http://grafana:3000/`.
  2. Cek log dengan perintah `docker logs --tail 50 grafana` dan `docker logs --tail 50 grafana-renderer`.
