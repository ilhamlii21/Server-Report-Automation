# Dokumen Riset: Automasi Laporan Bulanan Server Menggunakan Grafana

**Tujuan:** Mengotomatisasi pembuatan laporan bulanan server yang bersumber dari data Grafana, dengan menggunakan n8n sebagai orchestrator dan HolmesGPT (opsional) untuk analisis serta narasi otomatis.

**Tenggat Waktu:** Akhir bulan ini

**Referensi:**
- HolmesGPT: https://github.com/HolmesGPT/holmesgpt
- HolmesGPT Grafana Toolset: https://holmesgpt.dev/latest/data-sources/builtin-toolsets/grafanadashboards/

---

## Status Awal (Telah Selesai ✅)

- [x] Layanan Grafana, Prometheus, dan node-exporter telah beroperasi pada VM Azure (Spesifikasi: 2 vCPU / 4GB RAM).
- [x] Grafana Image Renderer telah terpasang dan berfungsi dengan baik (dideploy via Docker dengan konfigurasi variabel lingkungan `AUTH_TOKEN` — bukan `RENDERING_AUTH_TOKEN`).
- [x] Proses rendering manual melalui Antarmuka Pengguna (UI) Grafana (Share → Link → Generate image) berhasil dilakukan.
- [x] Service Account beserta Token autentikasi telah dibuat pada Grafana dengan hak akses (role): Viewer.
- [x] API Rendering telah berhasil diuji menggunakan perintah `curl` dan menghasilkan file PNG yang valid.
- [x] Daftar ID Panel pada dashboard `grafanavm2` (UID: `rYdddlPWk`) telah diidentifikasi dan dicatat.

---

## Arsitektur Target

```
[n8n Cron Trigger: Setiap tanggal 1 pukul 00:00]
        |
        v
[HTTP Request] --> Grafana Render API (Pengambilan gambar tiap panel, rentang waktu = bulan lalu)
        |
        v
[HTTP Request] --> Prometheus Query API (Opsional, pengambilan metrik numerik: rata-rata/maksimum/persentase uptime)
        |
        v
[HTTP Request / Execute Command] --> HolmesGPT atau LLM API (Analisis & pembuatan narasi otomatis)
        |
        v
[Code Node] --> Penyusunan dokumen laporan dalam format HTML/Markdown sesuai template
        |
        v
[Execute Command / HTML-to-PDF] --> Konversi dokumen ke format PDF
        |
        v
[Email / Slack Node] --> Distribusi laporan kepada tim terkait
```

---

## Minggu 1 — Fondasi & Analisis Kebutuhan

### Step 1.1 — Template Laporan Bulanan (Telah Diperoleh)

Template resmi yang digunakan terdiri dari 11 bagian utama: Metadata, Executive Summary, System Resource Overview (CPU/Memory/Disk/Network), Service & Process Health (Systemd/PM2/Docker/Cron), Security Audit, Log Management, Cost Monitoring, Change Log, Recommendations, dan Attachments.

### Step 1.1b — Pemetaan Template ke Sumber Data & Penentuan Prioritas

| Bagian Laporan | Sumber Data | Tingkat Prioritas (Tier) | Status |
| --- | --- | --- | --- |
| 1. Report Metadata | Statis / Variabel n8n | Tier 1 | Dapat digenerate secara otomatis |
| 2. Executive Summary | Narasi AI (HolmesGPT/LLM), dibuat setelah seluruh data pendukung terkumpul | Tier 1 | Otomatisasi setelah data utama siap |
| 3.1 CPU Usage | Prometheus `node_cpu_seconds_total` | Tier 1 | ✅ Siap digunakan |
| 3.2 Memory + Swap | Prometheus `node_memory_*` | Tier 1 | ✅ Siap digunakan |
| 3.2 Top proses RAM | Memerlukan `process-exporter` (node-exporter standar tidak mencatat metrik per proses) | Tier 2 | Memerlukan pemasangan exporter tambahan |
| 3.3 Disk Usage | Prometheus `node_filesystem_*` | Tier 1 | ✅ Siap digunakan |
| 3.3 Top 5 folder terbesar | Tidak tersedia di exporter standar — memerlukan skrip custom `du -sh` + Pushgateway | Tier 3 | Memerlukan skrip kustom |
| 3.4 Network Usage | Prometheus `node_network_*` | Tier 1 | ✅ Siap digunakan |
| 4.1 Systemd Services | node-exporter systemd collector (secara bawaan nonaktif, perlu `--collector.systemd`) | Tier 2 | Memerlukan aktivasi collector |
| 4.2 PM2 | Tidak tersedia secara bawaan — memerlukan `pm2-prometheus-exporter` atau parsing `pm2 jlist` | Tier 2/3 | Memerlukan perangkat tambahan (opsional) |
| 4.3 Docker Health | Memerlukan **cAdvisor** (metrik kontainer), tidak tercakup dalam node-exporter | Tier 2 | Memerlukan pemasangan exporter tambahan |
| 4.4 Cronjob Audit | Tidak tersedia di exporter standar — parsing `crontab -l` + log cron melalui skrip | Tier 3 | Memerlukan skrip kustom |
| 5. Security Audit | Log SSH via Loki+Promtail, kedaluwarsa SSL via `blackbox_exporter`, update via `apt list` | Tier 3 | Memerlukan infrastruktur tambahan (Loki) |
| 6. Log Management | Loki + Promtail untuk agregasi log dan deteksi lonjakan anomali | Tier 3 | Memerlukan infrastruktur tambahan |
| 7. Cost Monitoring | **Azure Cost Management API** (sumber data terpisah sepenuhnya dari Grafana) | Tier 3 | Integrasi API pihak ketiga |
| 9. Change Log | Pengisian manual atau otomatisasi melalui git log (jika menerapkan Infrastructure as Code) | Tier 3 | Kemungkinan tetap diisi secara manual |
| 10. Recommendations | Narasi AI berdasarkan analisis pola dari seluruh data di atas | Tier 1 | Menggunakan mekanisme yang sama dengan Executive Summary |
| 11. Attachments | Hasil tangkapan layar (screenshot) via Grafana Render API | Tier 1 | ✅ Siap digunakan |

**Strategi Implementasi:** Fokus pengembangan Minimum Viable Product (MVP) diarahkan pada **Tier 1** (proses otomatisasi penuh dengan infrastruktur yang sudah ada). Integrasi untuk **Tier 2** dan **Tier 3** akan didokumentasikan sebagai rencana pengembangan jangka panjang (roadmap) yang memerlukan keputusan manajemen serta instalasi komponen tambahan.

- [ ] Melakukan koordinasi dengan tim/stakeholder untuk memastikan apakah PM2 digunakan pada lingkungan produksi (jika tidak, bagian 4.2 dapat diabaikan).
- [ ] Melakukan koordinasi dengan tim/stakeholder mengenai prioritas pengembangan fitur Tier 2/3 setelah fase MVP Tier 1 selesai diimplementasikan.

### Step 1.2 — Konfigurasi Service Account & Token Grafana

1. Akses menu: Administration → Users and access → Service accounts → Add service account.
2. Tetapkan peran (role): **Viewer**.
3. Pilih opsi: Add service account token, kemudian simpan token secara aman (hindari melakukan commit token ke repositori git).

### Step 1.3 — Eksplorasi Lanjutan API Rendering

Metode yang telah teruji dan berhasil:

```bash
curl -s -H "Authorization: Bearer <SERVICE_ACCOUNT_TOKEN>" \
  "http://localhost:3000/render/d-solo/<DASHBOARD_UID>/<slug>?orgId=1&panelId=<PANEL_ID>&width=1000&height=500&from=<FROM>&to=<TO>&tz=Asia%2FMakassar" \
  -o output.png
```

**Aspek yang memerlukan eksplorasi lebih lanjut:**
- [ ] Menguji rendering dengan parameter rentang waktu `from` dan `to` yang presisi sesuai kalender bulanan (menggunakan Unix timestamp milidetik).
- [ ] Menguji kelayakan rendering untuk **keseluruhan dashboard** secara langsung (tanpa memisah per panel) melalui endpoint `/render/d/<uid>`.
- [ ] Menguji pemanggilan langsung ke API Prometheus untuk mendapatkan metrik mentah (raw data) guna menyusun tabel ringkasan:
  ```bash
  curl -s "http://localhost:9090/api/v1/query_range?query=<promql>&start=<ts>&end=<ts>&step=3600"
  ```
- [ ] Menetapkan daftar final ID Panel yang relevan untuk laporan bulanan:
  | ID Panel | Deskripsi Metrik |
  |---|---|
  | 77 | CPU Basic |
  | 78 | Memory Basic |
  | 74 | Network Traffic Basic |
  | 152 | Disk Space Used Basic |
  | 15 | Uptime |

### Checklist Minggu 1
- [ ] Template laporan telah dianalisis secara menyeluruh.
- [ ] Daftar final panel dan metrik yang diperlukan untuk laporan telah ditetapkan.
- [ ] Skrip otomatisasi rendering multi-panel (looping) berbasis bash dapat dieksekusi dengan stabil.

---

## Minggu 2 — Konfigurasi n8n & Workflow Dasar

### Step 2.1 — Instalasi n8n (Rekomendasi: Self-hosted via Docker)

```bash
mkdir -p ~/n8n-data
docker run -d --name n8n \
  -p 5678:5678 \
  -v ~/n8n-data:/home/node/.n8n \
  -e GENERIC_TIMEZONE="Asia/Makassar" \
  -e TZ="Asia/Makassar" \
  --restart unless-stopped \
  docker.n8n.io/n8nio/n8n
```
Akses sistem: `http://<IP-VM>:5678`

**Alternatif:** Registrasi akun uji coba n8n Cloud (14 hari) menggunakan akun instansi/pribadi untuk melakukan evaluasi perbandingan antarmuka dan performa dengan opsi self-hosted.

### Step 2.2 — Pembuatan Workflow Dasar: Render Tunggal via n8n

Kebutuhan node pada n8n:
1. **Manual Trigger** (untuk pengujian, nantinya digantikan oleh Cron Trigger).
2. **HTTP Request Node**:
   - Metode: GET
   - URL: `http://grafana:3000/render/d-solo/<UID>/<slug>` (jika n8n berada dalam jaringan Docker yang sama dengan Grafana) atau `http://<IP-VM>:3000/render/...` (jika berbeda host).
   - Parameter Query: `orgId`, `panelId`, `width`, `height`, `from`, `to`, `tz`.
   - Header: `Authorization: Bearer <SERVICE_ACCOUNT_TOKEN>`.
   - Format Respon: **File** (karena output berupa data biner PNG).
3. **Write Binary File Node**: Menyimpan hasil render ke penyimpanan lokal atau meneruskannya ke langkah selanjutnya sebagai data biner.

- [ ] Memastikan keberhasilan rendering 1 panel utama dari n8n.
- [ ] Memverifikasi bahwa gambar hasil rendering n8n identik dengan hasil eksekusi manual via curl.

### Step 2.3 — Implementasi Loop untuk Multi-Panel

- [ ] Menggunakan node **Loop Over Items (Split in Batches)** dengan input berupa array daftar panel (ID dan nama).
- [ ] Mengeksekusi pemanggilan HTTP Request pada setiap iterasi dengan parameter `panelId` dinamis.
- [ ] Mengumpulkan seluruh hasil render ke dalam koleksi data biner yang terpadu.

### Step 2.4 — Otomatisasi Parameterisasi Rentang Waktu (Bulan Sebelumnya)

Menggunakan ekspresi JavaScript pada n8n:
```js
{{ $now.startOf('month').minus({months: 1}).toMillis() }}   // Waktu Mulai (FROM)
{{ $now.startOf('month').minus({seconds: 1}).toMillis() }}  // Waktu Akhir (TO)
```
- [ ] Memastikan ekspresi di atas menghasilkan timestamp yang valid dan presisi untuk rentang waktu bulan sebelumnya.

### Checklist Minggu 2
- [ ] n8n telah terinstal dan dapat diakses dengan baik.
- [ ] Workflow untuk rendering satu panel berhasil diimplementasikan.
- [ ] Workflow rendering multi-panel (looping) berjalan dengan baik.
- [ ] Sistem otomatisasi perhitungan rentang tanggal bulan sebelumnya berfungsi secara akurat.

---

## Minggu 3 — Otomatisasi Narasi (HolmesGPT) & Penyusunan Dokumen

### Step 3.1 — Instalasi & Konfigurasi HolmesGPT (Uji Coba Terpisah)

```bash
pip install holmesgpt --break-system-packages
```
Konfigurasi file `~/.holmes/config.yaml`:
```yaml
toolsets:
  grafana/dashboards:
    enabled: true
    config:
      api_key: <SERVICE_ACCOUNT_TOKEN>
      api_url: http://localhost:3000
      enable_rendering: true
```
Pengujian fungsionalitas:
```bash
holmes ask "Analisis panel CPU Basic dari dashboard grafanavm2 selama 30 hari terakhir, apakah ada anomali?"
```
- [ ] Memastikan HolmesGPT dapat terhubung ke instansi Grafana.
- [ ] Memastikan HolmesGPT berhasil merender dan menganalisis panel menggunakan kapabilitas Vision (memerlukan API Key dari penyedia LLM seperti OpenAI/Anthropic).
- [ ] Mengevaluasi kualitas narasi yang dihasilkan dari aspek akurasi dan kesesuaian untuk laporan formal.

### Step 3.2 — Integrasi HolmesGPT dengan n8n

Opsi integrasi yang dieksplorasi:
- [ ] **HolmesGPT HTTP API** (menjalankan HolmesGPT sebagai service/daemon) -> dihubungkan melalui node HTTP Request di n8n.
- [ ] **Execute Command Node**: Menjalankan perintah `holmes ask "..."` secara langsung pada terminal VM dan menangkap output teksnya.
- [ ] **Rencana Cadangan (Fallback)**: Jika integrasi HolmesGPT memerlukan waktu yang tidak mencukupi, pengiriman gambar panel dapat ditujukan langsung ke API LLM (Anthropic/OpenAI) via HTTP Request dengan prompt kustom.

### Step 3.3 — Penyusunan Dokumen Laporan

- [ ] Membuat **Code Node** di n8n untuk menyusun dokumen HTML menggunakan template (menyisipkan gambar panel dalam format Base64, teks narasi analitis dari LLM, dan metrik numerik dari Prometheus).
- [ ] Konversi HTML ke format PDF:
  - Opsi A: Menggunakan **Execute Command Node** untuk memanggil utilitas `wkhtmltopdf` (memerlukan instalasi di VM: `sudo apt install wkhtmltopdf`).
  - Opsi B: Menggunakan pustaka Puppeteer via skrip Node.js kustom.
  - Opsi C: Mengevaluasi penggunaan community node n8n untuk konversi HTML-to-PDF.

### Checklist Minggu 3
- [ ] HolmesGPT berhasil menghasilkan narasi analisis yang valid dari panel Grafana.
- [ ] Metode integrasi antara HolmesGPT dan n8n telah diputuskan dan diimplementasikan.
- [ ] Pembuatan template HTML laporan secara otomatis yang memuat gambar, narasi, dan angka metrik berhasil dilakukan.

---

## Minggu 4 — Integrasi End-to-End & Dokumentasi Akhir

### Step 4.1 — Penggabungan Alur Kerja Terintegrasi (End-to-End)

```
Cron Trigger (Tanggal 1 setiap bulan, pukul 00:00)
  -> Render seluruh panel via Grafana Render API
  -> Pengambilan metrik ringkasan via Prometheus API
  -> Pengiriman data ke HolmesGPT/LLM untuk penyusunan narasi analitis
  -> Penyusunan dokumen HTML berdasarkan template
  -> Konversi dokumen HTML menjadi file PDF
  -> Distribusi file PDF via Email/Slack ke tim penerima
```
- [ ] Memastikan alur kerja berjalan secara otomatis dari awal hingga akhir tanpa intervensi manual.
- [ ] Melakukan pengujian trigger manual untuk memastikan output dokumen telah sesuai spesifikasi sebelum mengaktifkan jadwal otomatis.
- [ ] Mengaktifkan Cron Trigger setelah sistem dipastikan stabil.

### Step 4.2 — Pengujian Keamanan & Validasi Sistem

- [ ] Membandingkan hasil laporan otomatis dengan laporan manual terdahulu guna memverifikasi akurasi data dan format dokumen.
- [ ] Melakukan simulasi kegagalan sistem (misal: Grafana tidak merespon, token autentikasi kedaluwarsa) untuk memastikan n8n mengirimkan notifikasi peringatan kegagalan.
- [ ] Memastikan aspek keamanan: seluruh kredensial disimpan pada fitur Credentials n8n (tidak tertulis langsung pada kode/node) dan tidak terekspos pada log sistem.

### Step 4.3 — Penyusunan Laporan Hasil Riset

Menyusun laporan akhir riset untuk diserahkan kepada pihak manajemen/stakeholder, meliputi:
- [ ] Arsitektur sistem akhir yang diterapkan (diagram beserta penjelasan komponen).
- [ ] Analisis perbandingan keberhasilan implementasi dan kendala teknis (misal: keterbatasan sumber daya VM, limitasi API rate limit, atau batasan kapasitas analisis HolmesGPT).
- [ ] Estimasi biaya operasional (jika menggunakan n8n Cloud) atau kebutuhan sumber daya komputasi (jika menerapkan self-hosted).
- [ ] Rekomendasi langkah menuju tahap produksi (monitoring kestabilan workflow n8n, mekanisme pencadangan konfigurasi, dll).
- [ ] Lampiran sampel dokumen laporan bulanan yang berhasil dihasilkan secara otomatis.

---

## Informasi Referensi Teknis

**Penyimpanan Kredensial (Sensitif - Hindari Commit ke Git/Repositori Publik):**
- Grafana Renderer Token: Autentikasi internal antara Grafana dan kontainer Image Renderer.
- Grafana Service Account Token: Autentikasi untuk pemanggilan API eksternal (curl, n8n, HolmesGPT).

**Informasi Dashboard Pengujian:**
- UID Dashboard: `rYdddlPWk`
- Slug Dashboard: `grafanavm2`
- URL Utama Grafana: `http://localhost:3000` (akses lokal VM) atau `http://<IP-VM>:3000` (akses luar VM/jaringan eksternal).

**Perintah CLI Berguna:**
```bash
# Menampilkan seluruh daftar dashboard yang tersedia
curl -s -H "Authorization: Bearer <TOKEN>" "http://localhost:3000/api/search" | python3 -m json.tool

# Menampilkan daftar panel di dalam dashboard tertentu
curl -s -H "Authorization: Bearer <TOKEN>" "http://localhost:3000/api/dashboards/uid/<UID>" | \
  python3 -c "import json,sys; d=json.load(sys.stdin); [print(p.get('id'),'-',p.get('title')) for p in d['dashboard']['panels']]"

# Menjalankan server HTTP lokal sementara untuk verifikasi gambar hasil render
python3 -m http.server 8888
```
