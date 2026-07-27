# Panduan Integrasi Laporan Docker Health & Storage (Revisi Aman)

Dokumen ini menjelaskan konfigurasi untuk mengisi poin **4.3 Docker Health** pada template laporan bulanan server di Notion secara aman menggunakan data metrik dari Prometheus dan penyimpanan grafis dari Supabase.

---

## 1. Konsep Kerja & Keamanan Baru

Untuk alasan keamanan dan pembatasan firewall VM, n8n **tidak diizinkan mengeksekusi CLI Command secara langsung** di host VM. Seluruh status kesehatan Docker diambil melalui query HTTP API ke **Prometheus** yang datanya di-supply secara berkala oleh:
*   **Cronjob Lokal VM** (`collect_docker_storage.sh`) yang mencatat status internal Docker Engine ke folder `node_exporter` Textfile Collector.

---

## 2. Batasan Teknis & Solusi Metrik Docker

Berikut adalah status terkini mengenai data yang bisa dikumpulkan oleh Prometheus untuk Laporan Docker Health:

### **A. Running Containers (Keterbatasan Nama & Uptime)**
*   **Masalah**: Prometheus/cAdvisor tidak dapat menampilkan daftar string nama kontainer dinamis beserta uptimenya (seperti `prometheus (Up 2 days)`) dengan mudah untuk disisipkan ke blok Notion tanpa manipulasi JSON yang sangat rumit.
*   **Solusi**: Sebagai gantinya, kita memantau **jumlah total kontainer yang sedang berjalan** saja (misal: *7 containers running*). Metrik ini dihitung secara berkala di VM menggunakan cronjob (`docker ps -q | wc -l`) dan disimpan dalam metrik kustom `node_docker_containers_running`.

### **B. Volume, Image, & Reclaimable Space**
Metrik kapasitas Docker Storage diekstrak dari perintah `docker system df` oleh cronjob lokal VM, dikonversi menjadi tipe data desimal bytes, lalu diekspor sebagai metrik berikut:
*   `node_docker_volumes_count` & `node_docker_volumes_size_bytes` (Jumlah & ukuran volume lokal).
*   `node_docker_images_count` & `node_docker_images_size_bytes` (Jumlah & ukuran image terunduh).
*   `node_docker_reclaimable_size_bytes` (Ukuran sampah Docker yang bisa dibersihkan).

n8n akan melakukan query PromQL untuk menarik metrik-metrik tersebut, lalu memformat satuannya ke format yang manusiawi (misal: GB atau MB) menggunakan JavaScript di n8n.

---

## 3. Alur Pengiriman Gambar Grafik (Image Render)

Untuk grafik performa visual (seperti Grafik utilisasi CPU/RAM dari Grafana):
1.  n8n mengambil gambar grafik format PNG langsung dari **Grafana Image Renderer** via HTTP GET.
2.  Gambar tersebut kemudian diunggah (*upload*) secara otomatis ke **Supabase Storage** sebagai bucket penyimpanan sementara karena **MinIO lokal VM masih dalam tahap uji coba dan belum berhasil dikonfigurasi** (terhambat masalah autentikasi browser autofill).
3.  Supabase mengembalikan tautan publik (*Public URL*) gambar, yang kemudian dikirimkan oleh n8n ke Notion API untuk ditempelkan di halaman laporan.

---

## 4. Query PromQL yang Digunakan n8n

Berikut adalah gabungan query PromQL yang dijalankan di n8n untuk mendapatkan data Docker Health:

```promql
{__name__=~"node_docker_.*"}
```

Respon JSON dari query di atas diproses oleh node Code n8n untuk menyusun string laporan yang dikirim ke Notion:
*   **Restart Count**: `node_docker_containers_running` (Metrik ini mengawasi apakah ada crash loop pada kontainer VM).
*   **Cleanup Status**: Membaca metrik `node_docker_reclaimable_size_bytes` untuk mendeteksi kapasitas memori yang terbuang sia-sia.
