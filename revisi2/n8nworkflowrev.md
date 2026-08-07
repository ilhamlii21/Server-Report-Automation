# Dokumentasi Workflow n8n: Server Report (Poin 1 - 11)

Dokumen ini memetakan alur kerja (workflow) n8n untuk pembuatan laporan bulanan/harian server, dimulai dari **Poin 1** hingga **Poin 11** (Attachment & Analisis Grafik) sesuai dengan struktur laporan pada Notion.

---

## **1. Report Metadata**
* **Jenis Node:** Notion Node (`3.6` / `n8n-nodes-base.notion`)
* **Block ID/Endpoint:** `https://api.notion.com/v1/blocks/poin1/children`
* **Fields:**
  * `Report Date`: `{{ $('Edit Fields').first().json.report_date }}` (format tanggal saat ini, contoh: `07 August 2026`)
  * `Period`: `{{ $('Edit Fields').first().json.period }}` (format bulan pelaporan dari H-1, contoh: `August 2026`)
  * `Server`: `{{ $('Edit Fields').first().json.server }}` (berisi nama VM, contoh: `ilhamvm2`)
  * `Reported By`: `{{ $('Edit Fields').first().json.reported_by }}` (berisi `n8n Server Automation`)
  * `PIC`: `{{ $('Edit Fields').first().json.pic }}` (berisi penanggung jawab, contoh: `Ilham`)
* **Notes:** Node ini digunakan untuk memperbarui metadata umum di halaman Notion. Tujuannya adalah memberikan informasi dasar di bagian awal laporan mengenai kapan laporan dibuat, periode audit, VM yang dipantau, PIC penanggung jawab, dan agen yang melaporkannya.

---

## **2. Executive Summary**
* **Jenis Node:**
  * **Merge Node (`Merge8`)**: Berfungsi menggabungkan seluruh hasil pembacaan metrik server lainnya (CPU, RAM, Disk, Jaringan, PM2, Cron, dsb.) sebelum diproses ke AI.
  * **Wait Node (`Wait12`)**: Jeda waktu selama 12 detik untuk memastikan sinkronisasi data masukan berjalan sempurna.
  * **HTTP Request Node (`HTTP Request12` / Analisis AI)**: Memanggil Layanan AI (`https://api.xxx/v1/chat/completions`) dengan model `gpt-5.6-sol` untuk memproses ringkasan eksekutif 4 poin terpisah tanpa karakter bold (`**`).
  * **HTTP Request Node (`3.12` / Notion Patch)**: Mengirimkan hasil analisis AI menggunakan metode `PATCH` ke Notion Block API (`https://api.notion.com/v1/blocks/poin2/children`).
* **Fields / Struktur Output:**
  * `Kondisi Server`: Rangkuman singkat kesehatan dasar CPU, RAM, dan Disk.
  * `Anomaly Issue`: Status anomali (seperti *zombie processes* atau IP mencurigakan).
  * `Perubahan Signifikan`: Riwayat update OS atau pembaruan konfigurasi penting dibanding bulan lalu.
  * `Action Bulan ini`: Langkah tindak lanjut utama yang direkomendasikan untuk DevOps.
* **Notes:** Node ini memproses kompilasi seluruh metrik server melalui AI untuk menghasilkan kesimpulan ringkas (*Executive Summary*) setebal 4 poin terpisah, lalu mempublikasikannya ke blok Notion yang bersangkutan.

---

## **3. System Resource Performance**
Bagian ini memantau empat pilar utama sumber daya server: CPU, Memory (RAM), Disk, dan Network.

### **3.1 CPU Performance**
* **Jenis Node:**
  * **Prometheus HTTP Request (`Prometheus AVG CPU`)**:
    * **Query:** `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1d])) * 100)`
    * **Deskripsi:** Mengukur rata-rata penggunaan CPU selama 1 hari terakhir.
  * **Prometheus HTTP Request (`Prometheus PEAK CPU`)**:
    * **Query:** `max_over_time((100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100))[1d:1m])`
    * **Deskripsi:** Mengukur puncak beban CPU (peak) dengan resolusi sampling 1 menit dalam 1 hari terakhir.
  * **HTTP Request (`HTTP Request5` / Analisis AI)**: Memanggil Layanan AI (`https://api.xxx/v1/chat/completions` - `gpt-5.4`) untuk membuat deskripsi analisis CPU dalam 1-2 kalimat.
  * **Notion Node (`3.1` / `n8n-nodes-base.notion`)**:
    * **Block ID:** `https://api.notion.com/v1/blocks/poin3.1/children`
    * **Fields:**
      * `Avg CPU`: Menampilkan angka rata-rata CPU terpakai dalam persen.
      * `Peak CPU`: Menampilkan angka penggunaan tertinggi CPU dalam persen.
      * `Notes`: Berisi teks analisis CPU dari AI.
* **Notes:** Node ini mengambil metrik CPU rata-rata dan puncak dari Prometheus, mengirimkannya ke AI untuk dianalisis, lalu memperbarui bagian performa CPU di Notion.

### **3.2 RAM Performance (Memory & Swap)**
* **Jenis Node:**
  * **Prometheus HTTP Request (`Prometheus Prometheus AVG RAM`)**:
    * **Query:** `avg_over_time(((1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100)[1d:5m])`
    * **Deskripsi:** Rata-rata penggunaan RAM (%) selama 24 jam terakhir.
  * **Prometheus HTTP Request (`Prometheus Peak RAM`)**:
    * **Query:** `max_over_time(((1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100)[1d:5m])`
    * **Deskripsi:** Puncak penggunaan RAM (%) selama 24 jam terakhir.
  * **Prometheus HTTP Request (`Prometheus Swap Used`)**:
    * **Query:** `max_over_time(((node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes) / 1024 / 1024 / 1024)[1d:5m])`
    * **Deskripsi:** Puncak penggunaan swap space (GB) selama 24 jam terakhir.
  * **Prometheus HTTP Request (`Prometheus Swap Total`)**:
    * **Query:** `node_memory_SwapTotal_bytes / 1024 / 1024 / 1024`
    * **Deskripsi:** Total kapasitas swap space (GB) yang terpasang di VM.
  * **Prometheus HTTP Request (`Prometheus Proses Makan RAM`)**:
    * **Query:** `topk(5, (sum(namedprocess_namegroup_memory_bytes{memtype="resident"}) by (groupname) / scalar(node_memory_MemTotal_bytes)) * 100)`
    * **Deskripsi:** Mengambil 5 grup proses dengan konsumsi RAM residu (resident memory) terbesar terhadap total memori.
  * **HTTP Request (`HTTP Request3` / Analisis AI)**: Meminta Layanan AI (`https://api.xxx/v1/chat/completions` - `gpt-5.4`) menganalisis RAM, Swap, dan proses teratas.
  * **HTTP Request (`HTTP Request Notion` / Notion Patch)**:
    * **Block ID:** `https://api.notion.com/v1/blocks/poin3.2/children`
    * **Fields:**
      * `Average RAM usage`
      * `Peak RAM usage`
      * `Swap usage` (Used vs Total Swap)
      * `Process paling makan RAM` (Daftar 5 proses teratas dalam format code block)
      * `Notes`: Analisis memori dari AI.
* **Notes:** Bagian ini memantau utilitas RAM fisik, Swap virtual, dan mengidentifikasi 5 proses aplikasi yang paling banyak memakan RAM untuk kemudian dilaporkan ke Notion setelah melalui analisis AI.

### **3.3 Disk Capacity & Usage**
* **Jenis Node:**
  * **Prometheus HTTP Request (`Prometheus Disk Usage (% / Partisi Root)`)**:
    * **Query:** `(1 - (node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100`
    * **Deskripsi:** Mengukur persentase pemakaian disk partisi root (`/`).
  * **Prometheus HTTP Request (`Prometheus Pertumbuhan Disk`)**:
    * **Query:** `((node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"}) - (node_filesystem_size_bytes{mountpoint="/"} offset 30d - node_filesystem_free_bytes{mountpoint="/"} offset 1d)) / 1024 / 1024 / 1024`
    * **Deskripsi:** Selisih volume pemakaian disk (GB) dibanding 30 hari yang lalu.
  * **Prometheus HTTP Request (`Prometheus Top 5 Folder Terbesar`)**:
    * **Query:** `topk(5, node_filesystem_size_bytes - node_filesystem_free_bytes)`
    * **Deskripsi:** Mengambil data ukuran penyimpanan dari 5 folder/mountpoint terbesar di server.
  * **HTTP Request (`HTTP Request4` / Analisis AI)**: Meminta Layanan AI (`https://api.xxx/v1/chat/completions` - `gpt-5.4-mini`) menganalisis ruang disk, laju pertumbuhan, dan folder terbesar.
  * **HTTP Request (`3.3` / Notion Patch)**:
    * **Block ID:** `https://api.notion.com/v1/blocks/poin3.3/children`
    * **Fields:**
      * `Disk % per partition`
      * `Pertumbuhan disk dibanding bulan lalu`
      * `Folder Terbesar (Top 5)` (dimasukkan dalam callout block gray dengan ikon 📁)
      * `Notes`: Analisis disk dari AI.
* **Notes:** Memantau kapasitas disk pada partisi root `/`, laju pertumbuhan data bulanan, dan mencari folder yang menggunakan ruang paling banyak guna mencegah kehabisan kapasitas (*disk full*).

### **3.4 Network Traffic Performance**
* **Jenis Node:**
  * **Prometheus HTTP Request (`Prometheus NET Inbound`)**:
    * **Query:** `sum(rate(node_network_receive_bytes_total{device!~"lo"}[1d])) * 8 / 1000000`
    * **Deskripsi:** Rata-rata bandwidth masuk (RX) dalam Mbps pada seluruh interface aktif (kecuali loopback) selama 1 hari terakhir.
  * **Prometheus HTTP Request (`Prometheus NET Outbond`)**:
    * **Query:** `sum(rate(node_network_transmit_bytes_total{device!~"lo"}[1d])) * 8 / 1000000`
    * **Deskripsi:** Rata-rata bandwidth keluar (TX) dalam Mbps pada seluruh interface aktif (kecuali loopback) selama 1 hari terakhir.
  * **HTTP Request (`HTTP Request` / Analisis AI)**: Memanggil Layanan AI (`https://api.xxx/v1/chat/completions` - `gpt-5.4`) untuk menganalisis statistik inbound/outbound jaringan.
  * **Notion Node (`3.4` / `n8n-nodes-base.notion`)**:
    * **Block ID:** `https://api.notion.com/v1/blocks/poin3.4/children`
    * **Fields:**
      * `Average RX/TX`: Mengisi rata-rata kecepatan RX dan TX dalam Mbps.
      * `Notes`: Analisis traffic jaringan dari AI.
* **Notes:** Mengambil statistik rata-rata traffic jaringan inbound dan outbound server, meminta AI mengevaluasi beban traffic tersebut, lalu menuliskannya ke Notion.

---

## **4. Service & Processes Health**
Bagian ini mengaudit kesehatan layanan-layanan penting sistem, status PM2 (Node.js), kontainer Docker, dan audit berkala Cronjob.

### **4.1 Systemd & Zombie Processes**
* **Jenis Node:**
  * **Prometheus HTTP Request Node (`Systemd`)**:
    * **Query:** `({__name__='node_systemd_unit_state', state='active'} == 1) or ({__name__='node_systemd_unit_state', state='failed'} == 1) or {__name__='node_processes_zombies'}`
    * **Deskripsi:** Mengambil data status keaktifan unit systemd (active/failed) dan metrik zombie processes dalam satu query sekaligus.
  * **Code Node (`Code in JavaScript`)**:
     * **Bahasa:** JavaScript
     * **Deskripsi:** Memproses output data mentah dari node Prometheus:
       - Menyaring unit dengan status `active` dan menghitung jumlahnya (mengambil 5 contoh service pertama).
       - Menyaring unit dengan status `failed` dan menghitung jumlahnya beserta nama unitnya.
       - Mendapatkan nilai dari metrik zombie process (`node_processes_zombies`).
       - Memformat data menjadi format baris teks gabungan (`stdoutText`):
         `ACTIVE:<jumlah> (Contoh: ...) | FAILED:<jumlah> (Detail: ...) | ZOMBIE:<jumlah>`
  * **Wait Node (`Wait4`)**: Node jeda untuk sinkronisasi antrean pemrosesan.
  * **HTTP Request Node (`HTTP Request1` / Analisis AI)**:
    * Memanggil Layanan AI (`https://api.xxx/v1/chat/completions`) dengan model `gpt-5.4-mini` untuk merumuskan analisis teknis 1-2 kalimat dalam Bahasa Indonesia mengenai kesehatan service systemd dan zombie process berdasarkan input teks terformat.
  * **Notion Node (`4.1` / `n8n-nodes-base.notion`)**:
    * **Block ID:** `https://api.notion.com/v1/blocks/poin4.1/children`
    * **Fields:**
      * `Active / Failed`: Mengisi jumlah service aktif vs gagal, diambil dari parsing string output JavaScript:
        `{{ $('Code in JavaScript').first().json.stdout.split('|')[0].replace('ACTIVE:', '').trim() }} / {{ $('Code in JavaScript').first().json.stdout.split('|')[1].replace('FAILED:', '').trim() }}`
      * `Zombie processes`: Mengisi nilai zombie processes dari string output JavaScript:
        `{{ $('Code in JavaScript').first().json.stdout.split('|')[2].replace('ZOMBIE:', '').trim() }}`
      * `Notes`: Mengisi hasil analisis AI.
* **Notes:** Poin ini mengawasi kesehatan service systemd pada server (memastikan tidak ada service penting yang mati/failed) serta memastikan tidak ada penumpukan proses zombie yang membebani kernel sistem.

### **4.2 PM2 Health**
* **Jenis Node:**
  * **Prometheus HTTP Request Node (`Prometheus PM2 List services`)**:
    * **Query:** `pm2_up`
    * **Deskripsi:** Mengambil status operasional dari PM2 instance (1 untuk online, 0 untuk offline).
  * **Prometheus HTTP Request Node (`Prometheus PM2 Restart1`)**:
    * **Query:** `pm2_uptime`
    * **Deskripsi:** Mendapatkan lama waktu aktif (uptime) dari masing-masing process PM2 dalam satuan detik.
  * **Prometheus HTTP Request Node (`Prometheus PM2 Restart`)**:
    * **Query:** `pm2_restarts`
    * **Deskripsi:** Mengambil total jumlah restart count dari masing-masing process PM2.
  * **Merge Node (`Merge6`)**: Menggabungkan output dari ketiga query metrik PM2 (`pm2_up`, `pm2_uptime`, `pm2_restarts`).
  * **Wait Node (`Wait5`)**: Memberikan jeda sinkronisasi data sebelum diteruskan ke AI.
  * **HTTP Request Node (`HTTP Request2` / Analisis AI)**:
    * Memanggil Layanan AI (`https://api.xxx/v1/chat/completions`) dengan model `gpt-5.4-mini` untuk menganalisis performa PM2 dan menuliskan analisis singkat dalam Bahasa Indonesia yang menyebutkan status service, jumlah restart, dan uptime dalam hari.
  * **HTTP Request Node (`3.` / Notion Patch)**:
    * **Block ID/Endpoint:** `https://api.notion.com/v1/blocks/poin4.2/children`
    * **Fields:**
      * `List services (online / stopped / errored)`: Daftar nama service PM2 beserta statusnya (dalam format Code Block).
      * `Restart count`: Total frekuensi restart masing-masing service PM2.
      * `Uptime`: Durasi aktif masing-masing service PM2 yang telah dikonversi ke satuan Hari.
      * `Notes`: Catatan analisis dari AI.
* **Notes:** Memantau seluruh aplikasi Node.js yang dikelola oleh PM2 (seperti webhook, server, dll.) untuk menjamin aplikasi berjalan tanpa mengalami crash berulang (*flapping*).

### **4.3 Docker Health**
* **Jenis Node:**
  * **Prometheus HTTP Request Node (`Docker List Cadvisor`)**:
    * **Query:** `time() - container_start_time_seconds{name!=""}`
    * **Deskripsi:** Menghitung durasi uptime dari masing-masing kontainer Docker yang aktif (berdasarkan selisih waktu Unix saat ini dengan waktu mulai kontainer).
  * **Prometheus HTTP Request Node (`Docker Size`)**:
    * **Query:** `container_memory_usage_bytes{name!=""} / 1024 / 1024`
    * **Deskripsi:** Menghitung penggunaan memori RAM per kontainer Docker dalam satuan MB.
  * **Merge Node (`Merge`)**: Menggabungkan data dari query Uptime kontainer dan Memory Usage kontainer.
  * **Code Node (`Code in JavaScript1`)**:
    * **Bahasa:** JavaScript
    * **Deskripsi:** Memetakan RAM dan Uptime berdasarkan nama kontainer, mengonversi uptime ke format "Up X days" atau "Up X hours", menghitung total kontainer yang aktif, serta memformat output untuk keperluan AI (`container_text_ai`), Notion Box Abu-Abu (`container_box_notion`), dan status restart (`restart_list`).
  * **Wait Node (`Wait11`)**: Memberikan jeda sinkronisasi antrean.
  * **HTTP Request Node (`HTTP Request14` / Analisis AI)**:
    * Memanggil Layanan AI (`https://api.xxx/v1/chat/completions`) dengan model `gpt-5.4-mini` untuk mengevaluasi kesehatan Docker daemon dan kontainer server, menghasilkan paragraf analisis dalam Bahasa Indonesia diawali kata "Notes: ".
  * **HTTP Request Node (`3.13` / Notion Patch)**:
    * **Block ID/Endpoint:** `https://api.notion.com/v1/blocks/poin4.3/children`
    * **Fields:**
      * `Running containers`: Jumlah container aktif (contoh: "11 container aktif").
      * `Container Box Notion`: Daftar kontainer, uptime status (healthy), dan memori terpakai (dalam format Code Block).
      * `Restart count abnormal?`: Daftar status restart per kontainer (default: `0x restart`).
      * `Notes`: Hasil tulisan analisis dari AI.
* **Notes:** Memonitor kontainer-kontainer Docker yang berjalan di server lewat metrik cAdvisor, memetakan konsumsi memorinya masing-masing, serta melacak kestabilan uptime untuk mendeteksi kontainer yang bermasalah.

### **4.4 Cronjob Audit**
* **Jenis Node:**
  * **Prometheus HTTP Request Node (`Cron`)**:
    * **Query:** `cron_job_executions_total`
    * **Deskripsi:** Mengambil total jumlah eksekusi cron job per user yang terekam pada metrik server.
  * **Wait Node (`Wait7`)**: Memberikan jeda sinkronisasi antrean.
  * **HTTP Request Node (`HTTP Request7` / Analisis AI)**:
    * Memanggil Layanan AI (`https://api.xxx/v1/chat/completions`) dengan model `gpt-5.4-mini` untuk mengolah data eksekusi cronjob mentah dan meresponsnya dalam format JSON terstruktur: `{"cron_list": "...", "failing_status": "...", "schedule_change": "...", "notes": "..."}`.
  * **HTTP Request Node (`3.7` / Notion Patch)**:
    * **Block ID/Endpoint:** `https://api.notion.com/v1/blocks/poin4.4/children`
    * **Fields:**
      * `Cronjob list`: Daftar user dan jumlah eksekusinya (dalam format Code Block).
      * `Cronjob failing?`: Status kegagalan/error cron job berdasarkan JSON dari AI.
      * `Perubahan jadwal?`: Status perubahan jadwal (schedule change) berdasarkan JSON dari AI.
      * `Notes`: Ringkasan analisis cronjob keseluruhan dari AI.
* **Notes:** Memantau aktivitas cronjob di latar belakang server untuk memastikan semua tugas terjadwal berjalan sesuai waktu yang semestinya dan mendeteksi adanya kegagalan eksekusi.

---

## **5. Security Audit**
* **Jenis Node:**
  * **Prometheus HTTP Request Node (`Security by Grok`)**:
    * **Query:** `ssh_login_success_total or ssh_login_fail_total`
    * **Deskripsi:** Mengambil data log audit login SSH yang sukses (`ssh_login_success_total`) dan gagal (`ssh_login_fail_total`) dari metrik Prometheus.
  * **Code Node (`Code Security`)**:
    * **Bahasa:** JavaScript
    * **Deskripsi:** Memproses output data login SSH dari Prometheus:
      - Menghitung akumulasi login sukses (`totalSuccess`) dan gagal (`totalFail`).
      - Menyaring IP client (`client_ip`) yang memiliki kegagalan login terbanyak untuk diidentifikasi sebagai IP mencurigakan (`suspiciousIp`).
      - Mengembalikan JSON hasil olah data: `ssh_login_summary` ("X Success / Y Fail"), `suspicious_ip`, `total_success`, dan `total_fail`.
  * **Wait Node (`Wait8`)**: Node jeda untuk sinkronisasi antrean pemrosesan data keamanan.
  * **HTTP Request Node (`AI Security` / Analisis AI)**:
    * Memanggil Layanan AI (`https://api.xxx/v1/chat/completions`) dengan model `gpt-5.6-sol` untuk menganalisis data login SSH dan menghasilkan 1-2 kalimat ringkas dalam Bahasa Indonesia yang diawali kata "Notes: " dan tanpa format markdown (`**`).
  * **HTTP Request Node (`3.8` / Notion Patch)**:
    * **Block ID/Endpoint:** `https://api.notion.com/v1/blocks/poin5/children`
    * **Fields:**
      * `SSH login (success / fail)`: Ringkasan status login SSH dari output javascript (contoh: "0 Success / 0 Fail").
      * `Suspicious IP`: IP address dengan upaya login gagal terbanyak (contoh: IP address penyerang atau "None").
      * `Notes`: Analisis keamanan SSH yang dihasilkan oleh AI.
* **Notes:** Poin ini mengaudit stabilitas dan keamanan akses SSH ke server untuk mendeteksi upaya brute force dan mencatat alamat IP luar yang mencurigakan.

---

## **6. Log Management**
* **Jenis Node:**
  * **Prometheus HTTP Request Node (`Log by Grok`)**:
    * **Query:** `syslog_errors_total`
    * **Deskripsi:** Mengambil total error sistem (`syslog_errors_total`) yang ditangkap grok_exporter dari log syslog server.
  * **Wait Node (`Wait9`)**: Node jeda untuk sinkronisasi antrean pemrosesan logs.
  * **HTTP Request Node (`HTTP Request9` / Analisis AI)**:
    * Memanggil Layanan AI (`https://api.xxx/v1/chat/completions`) dengan model `gpt-5.4-mini` untuk merumuskan analisis teknis 1-2 kalimat dalam Bahasa Indonesia mengenai manajemen log server (menyebutkan log spikes dan pola error terbanyak) tanpa format markdown (`**`).
  * **HTTP Request Node (`3.9` / Notion Patch)**:
    * **Block ID/Endpoint:** `https://api.notion.com/v1/blocks/poin6/children`
    * **Fields / Struktur Output:**
      * `Log spikes`: Status lonjakan log (Yes/No) berdasarkan total error (> 50 error dianggap "Yes" beserta jumlahnya).
      * `Error patterns`: Menampilkan 3 program dengan jumlah error tertinggi (contoh: "15x dockerd, 5x sshd").
      * `Notes`: Catatan analisis dari AI.
      * `6.1 Monthly Cleanup Tasks` (Heading 3): Daftar checklist cleanup bulanan:
        * [ ] Clean `/tmp`
        * [ ] Clean `/var/log` old logs
        * [ ] Docker prune
        * [ ] Remove unused backups
* **Notes:** Poin ini berfungsi untuk memantau aktivitas log server guna mengidentifikasi error aplikasi/sistem yang sering muncul, memantau *log spikes*, serta menyertakan daftar cleanup pemeliharaan berkala bulanan di Notion.

---

## **7. Cost Monitoring (Opsional Jika Cloud)**
* **Notes:** Poin ini bersifat opsional untuk pemantauan biaya infrastruktur jika server berada di layanan Cloud (seperti AWS, GCP, Azure, dsb.). Di dalam workflow n8n ini, pemantauan Cost/Biaya belum diimplementasikan karena pemantauan dilakukan pada infrastruktur lokal/on-premise (VM independen). Namun belum ada indikasi untuk memantau hal ini kedepannya.

---

## **8. -**
* **Notes:** Kosong / tidak digunakan.

---

## **9. Change Log (OS & Container Version Control)**
* **Jenis Node:**
  * **Prometheus HTTP Request Node (`OS by node`)**:
    * **Query:** `node_uname_info`
    * **Deskripsi:** Mengambil rincian metrik informasi sistem operasi, tipe arsitektur, dan versi rilis kernel server.
  * **Prometheus HTTP Request Node (`Docker Image by Cadvisor`)**:
    * **Query:** `count by (image) (container_last_seen{image!=""})`
    * **Deskripsi:** Mendapatkan daftar seluruh docker images yang aktif terdeteksi lewat cAdvisor.
  * **Merge Node (`Merge10`)**: Menggabungkan data versi OS dan daftar Docker images.
  * **Wait Node (`Wait10`)**: Node jeda sinkronisasi antrean.
  * **HTTP Request Node (`HTTP Request10` / Analisis AI)**:
    * Memanggil Layanan AI (`https://api.xxx/v1/chat/completions`) dengan model `gpt-5.6-sol` untuk menganalisis pembaruan/versi OS, kernel, dan daftar docker image yang aktif, lalu merumuskan Change Log teknis 1-2 kalimat dalam Bahasa Indonesia yang diawali kata "Notes: " tanpa format markdown (`**`).
  * **HTTP Request Node (`3.10` / Notion Patch)**:
    * **Block ID/Endpoint:** `https://api.notion.com/v1/blocks/poin9/children`
    * **Fields / Struktur Output:**
      * `OS update`: Informasi versi OS dan rilis kernel saat ini (contoh: "Linux Kernel 5.15.0-101-generic").
      * `Docker image aktif`: Daftar docker images yang sedang berjalan di sistem (digabungkan dengan tanda koma).
      * `Notes`: Catatan Change Log analisis dari AI.
* **Notes:** Poin ini mengawasi perubahan sistem operasi, kernel, serta versi kontainer Docker yang digunakan, yang berfungsi sebagai pengendali versi (*Version Control*) dan pencatatan riwayat perubahan (*Change Log*) lingkungan server.

---

## **10. DevOps Recommendations (Rekomendasi Tindakan DevOps)**
* **Jenis Node:**
  * **Wait Node (`Wait13`)**: Memberikan jeda sinkronisasi data masukan selama 10 detik sebelum diteruskan ke AI.
  * **HTTP Request Node (`HTTP Request11` / Analisis AI)**:
    * Memanggil Layanan AI (`https://api.xxx/v1/chat/completions`) dengan model `gpt-5.6-sol` untuk memproses seluruh metrik (CPU, RAM, Disk, Docker, Security, Logs, OS) dan menghasilkan 3-4 rekomendasi tindakan teknis DevOps untuk bulan depan, ringkas, padat (maks 250 kata), tanpa karakter markdown (`**`).
  * **HTTP Request Node (`3.11` / Notion Patch)**:
    * **Block ID/Endpoint:** `https://api.notion.com/v1/blocks/poin10/children`
    * **Fields / Struktur Output:**
      * `Rekomendasi Tindakan DevOps Bulan Depan:`: Bullet list header untuk Notion.
      * Isi Rekomendasi: Blok paragraf teks yang memuat poin-poin saran DevOps dari AI (dibatasi substring maks 1950 karakter agar aman di API Notion).
* **Notes:** Poin ini mengkompilasi seluruh hasil pemantauan kesehatan server dalam sebulan terakhir untuk dirumuskan menjadi daftar tindakan preventif/korektif DevOps di bulan mendatang.

---

## **11. Attachment & Analisis Grafik**
Bagian ini menangani pemotretan grafik performa dari Grafana, pengunggahan file gambar ke Supabase Storage, penyematan file gambar ke halaman Notion, serta analisis gambar visual tersebut secara multimodal menggunakan AI.

### **11.1 Attachment (Visual Grafik Grafana)**
* **Jenis Node:**
  * **HTTP Request Node untuk Grafana Render Image (`Get Image CPU`, `Get Image RAM`, `Get Image DISK`, `Get Image APPS RAM`)**:
    * Mengambil snapshot/rendering grafis panel langsung dari Grafana server (`http://20.205.17.18:3000`) dengan otentikasi Bearer API key Grafana.
    * *Contoh API URL (CPU):* `http://20.205.17.18:3000/render/d-solo/rYdddlPWk/grafanavm2?orgId=1&panelId=77&width=1000&height=500&from=now-30d&to=now&var-node=node-exporter:9100&var-job=node-exporter` (mengambil Panel ID `77` untuk grafis CPU).
    * Opsi respons diset format "File" (Binary).
  * **HTTP Request Node untuk Upload Supabase Storage (`HTTP Supabase Bucket`, `HTTP Supabase Bucket1`, `HTTP Supabase Bucket2`, `HTTP Supabase Bucket3`)**:
    * Mengunggah file biner gambar dari Grafana ke Supabase bucket (`grafanarenderiimage`) via Supabase REST API dengan nama file: `cpu_report.png`, `ram_report.png`, `disk_report.png`, dan `apps_ram_report.png` (menggunakan domain Supabase `https://xxx.supabase.co`).
    * Menggunakan metode `POST` dan otentikasi Bearer Service Role JWT Supabase, dengan header `x-upsert: true` untuk menimpa file lama.
  * **Merge Node (`Merge9`)**: Menggabungkan file-file gambar hasil upload Supabase.
  * **Notion Node (`Append a block1`)**:
    * **Block ID:** `https://api.notion.com/v1/blocks/poin11.1/children`
    * **Fungsi:** Menyematkan 4 tautan gambar Supabase (CPU, RAM, Disk, Apps RAM) ke Notion. Menyertakan parameter dinamis Unix Timestamp (`?t={{ $now.toUnixInteger() }}`) agar Notion memuat ulang gambar dari Supabase (anti-cache).
* **Daftar Visual Grafik Panel Grafana yang Diambil:**
  1. **CPU Basic Graphic** (Visual grafik utilitas CPU).
  2. **RAM Basic Graphic** (Visual grafik utilitas memori).
  3. **Disk Space Used Graphic** (Visual grafik kapasitas ruang disk).
  4. **Apps RAM Graphic** (Visual grafik konsumsi RAM per aplikasi).

### **11.2 Analisis Grafik oleh AI (AI - Vision)**
* **Jenis Node:**
  * **Wait Node (`Wait14`)**: Jeda waktu sebelum memproses AI untuk memastikan seluruh berkas gambar telah sukses diunggah ke Supabase.
  * **HTTP Request Node (`AI Analysis` / AI Vision Multimodal)**:
    * **URL:** `https://api.xxx/v1/chat/completions`
    * **Model:** `gpt-5.4-mini` (Model berkemampuan Visi/Multimodal).
    * **Prompt:** Mengirimkan 4 URL gambar Supabase di atas langsung ke model AI. AI diminta menganalisis performa server berdasarkan tren grafis visual tersebut dan menyajikan teks respons terstruktur menggunakan sub-kategori khusus (`1) CPU`, `2) RAM`, `3) Disk`, `4) Apps Ram`, dan `Ringkasan`).
  * **HTTP Request Node (`Notion` / Notion Patch)**:
    * **Block ID/Endpoint:** `https://api.notion.com/v1/blocks/poin11.2/children`
    * **Fungsi JavaScript Parser (Inline):**
      - Mengambil output konten AI dan memotong-motongnya menggunakan manipulasi index JavaScript berdasarkan kata kunci (`1) CPU`, `2) RAM`, dsb.).
      - Membatasi panjang teks per bagian maksimal 1900 karakter.
    * **Struktur Output Notion Toggle Blocks:**
      - Menuliskan teks pengantar
      - Menambahkan 5 blok **Toggle** Notion yang didalamnya berisi blok paragraf analisis detail:
        * **Toggle 1:** CPU Basic (Analisis)
        * **Toggle 2:** RAM Basic (Analisis)
        * **Toggle 3:** Disk Space Used (Analisis)
        * **Toggle 4:** Apps Ram (Analisis)
        * **Toggle 5:** Kesimpulan & Ringkasan Akhir
* **Notes:** Poin ini mengautomatiskan analisis pola visual grafik (misalnya mendeteksi lonjakan trafik atau penurunan memori bebas secara visual) dari dashboard Grafana menggunakan AI Vision, kemudian menyajikannya ke halaman Notion dalam bentuk toggles interaktif yang rapi.
