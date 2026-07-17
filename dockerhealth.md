# Panduan Automasi Laporan Docker Health Menggunakan n8n

Dokumen ini menjelaskan konfigurasi untuk mengisi poin **4.3 Docker Health** pada template laporan bulanan server di Notion secara otomatis menggunakan node **Execute Command** di n8n.

---

## 1. Konsep Kerja

Karena n8n terinstal secara native di VM (menggunakan PM2), n8n dapat berinteraksi langsung dengan Docker Daemon lokal melalui CLI. Kita menggunakan node **Execute Command** untuk mengeksekusi perintah CLI Docker, lalu memformat teks hasilnya menggunakan JavaScript di node **Code** untuk dikirim langsung ke Notion.

---

## 2. Pemetaan Command Docker & Kolom Notion

Berikut adalah perintah-perintah yang dijalankan pada node **Execute Command** di n8n untuk mendapatkan data di masing-masing kolom Notion:

### **A. Running Containers**
* **Tujuan**: Menampilkan daftar container yang sedang berjalan beserta uptime-nya.
* **Perintah CLI**:
  ```bash
  docker ps --format "{{.Names}} ({{.Status}})"
  ```
* **Contoh Output**:
  ```text
  prometheus (Up 13 minutes)
  grafana-renderer (Up 13 minutes (healthy))
  grafana (Up 13 minutes)
  ```

### **B. Restart Count Abnormal?**
* **Tujuan**: Mendeteksi apakah ada container yang sering mengalami crash/restart.
* **Perintah CLI**:
  ```bash
  docker inspect --format='{{.Name}}: {{.State.RestartCount}}' $(docker ps -aq)
  ```
* **Contoh Output**:
  ```text
  /prometheus: 0
  /grafana-renderer: 0
  /node-exporter: 2
  ```
* **Logika n8n**: Jika salah satu kontainer memiliki angka restart > 0, n8n akan menandainya sebagai `⚠ Ya (Terjadi Restart)` dan mencantumkan nama kontainernya. Jika semua bernilai 0, maka ditulis `✔ Tidak`.

### **C. Volume Size Summary**
* **Tujuan**: Mengambil ringkasan kapasitas memori disk yang digunakan oleh Docker Volume.
* **Perintah CLI**:
  ```bash
  docker system df | grep "Local Volumes" | awk '{print $4 " volumes (" $6 ")"}'
  ```
* **Contoh Output**:
  ```text
  4 volumes (1.2GB)
  ```

### **D. Image Size**
* **Tujuan**: Menampilkan total kapasitas penyimpanan yang digunakan oleh Docker Images yang terunduh.
* **Perintah CLI**:
  ```bash
  docker system df | grep "Images" | awk '{print $4 " images (" $6 ")"}'
  ```
* **Contoh Output**:
  ```text
  5 images (1.85GB)
  ```

### **E. Cleanup Status**
* **Tujuan**: Memeriksa apakah ada sampah image tak terpakai (*dangling*) yang belum dibersihkan.
* **Perintah CLI**:
  ```bash
  docker images -f dangling=true -q | wc -l
  ```
* **Logika n8n**: 
  * Jika output = `0`, status: `✔ Bersih (Tidak ada image dangling)`.
  * Jika output > `0`, status: `⚠ Perlu Cleanup (Ada X image menggantung)`.

---

## 3. Langkah-Langkah Pembuatan di n8n

### **Langkah 1: Membuat Node Execute Command**
1. Tambahkan node **Execute Command** di canvas n8n Anda.
2. Ubah kolom **Command** menjadi skrip gabungan berikut agar berjalan dalam satu panggilan:
   ```bash
   echo "===RUNNING===" && docker ps --format "{{.Names}} ({{.Status}})" && echo "===RESTARTS===" && docker inspect --format='{{.Name}}: {{.State.RestartCount}}' \$(docker ps -aq) && echo "===VOLUMES===" && docker system df | grep "Local Volumes" | awk '{print \$4 \" volumes (\" \$6 \")\"}' && echo "===IMAGES===" && docker system df | grep "Images" | awk '{print \$4 \" images (\" \$6 \")\"}' && echo "===DANGLING===" && docker images -f dangling=true -q | wc -l
   ```
3. Klik **Execute step** untuk mendapatkan output gabungan.

---

### **Langkah 2: Menambahkan Node Code (Parser JavaScript)**
Hubungkan output Execute Command ke node **Code** baru dengan script parser JavaScript berikut:

```javascript
const stdout = $('Execute Command').first().json.stdout;

// Parsing output gabungan
const runningSection = stdout.split('===RUNNING===\n')[1].split('===RESTARTS===')[0].trim();
const restartsSection = stdout.split('===RESTARTS===\n')[1].split('===VOLUMES===')[0].trim();
const volumesVal = stdout.split('===VOLUMES===\n')[1].split('===IMAGES===')[0].trim();
const imagesVal = stdout.split('===IMAGES===\n')[1].split('===DANGLING===')[0].trim();
const danglingVal = parseInt(stdout.split('===DANGLING===\n')[1].trim());

// Logika Deteksi Restart Abnormal
const restartLines = restartsSection.split('\n');
let abnormalRestarts = [];
for (const line of restartLines) {
  const parts = line.split(': ');
  if (parts.length === 2) {
    const name = parts[0].replace('/', '');
    const count = parseInt(parts[1]);
    if (count > 0) {
      abnormalRestarts.push(`${name} (${count}x)`);
    }
  }
}

const isAbnormal = abnormalRestarts.length > 0 
  ? `⚠ Ya (Terjadi Restart di: ${abnormalRestarts.join(', ')})` 
  : "✔ Tidak";

// Logika Cleanup Status
const cleanupStatus = danglingVal > 0 
  ? `⚠ Perlu Cleanup (Ada ${danglingVal} dangling image)` 
  : "✔ Bersih";

// Logika Notes Otomatis
const notes = abnormalRestarts.length > 0 
  ? "Harap periksa logs container yang sering restart." 
  : "Semua kontainer Docker berjalan optimal dan efisien.";

// Kirim ke Notion
return [{
  json: {
    running_containers: runningSection,
    restart_abnormal: isAbnormal,
    volume_size: volumesVal,
    image_size: imagesVal,
    cleanup_status: cleanupStatus,
    notes: notes
  }
}];
```

Dengan konfigurasi ini, seluruh poin **4.3 Docker Health** di Notion Anda akan terisi dengan data terbaru dari VM secara instan dan rapi!
