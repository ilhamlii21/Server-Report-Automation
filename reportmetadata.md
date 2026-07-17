# Panduan Konfigurasi Report Metadata Menggunakan n8n

Dokumen ini menjelaskan cara mengonfigurasi data metadata laporan (seperti Tanggal Laporan, Periode, Nama Server, Pembuat Laporan, dan PIC) secara otomatis menggunakan node **Edit Fields (Set)** di n8n untuk dikirimkan ke Notion.

---

## 1. Pendahuluan

Kolom **1. Report Metadata** pada template Notion berisi informasi administratif dasar. Data ini bersifat statis (diinput manual sekali saja) dan dinamis (dihitung otomatis menggunakan waktu server saat ini). Kita menggunakan node **Edit Fields (Set)** agar semua data ini terkumpul menjadi satu paket objek JSON sebelum dikirim ke database Notion.

---

## 2. Struktur Pengisian Node `Edit Fields` di n8n

Pada konfigurasi node **Edit Fields**, buatlah **5 field baru** dengan tipe data **String** (Teks) sebagai berikut:

### **1. Field 1: `report_date`**
* **Tipe Data**: `String`
* **Nama Field**: `report_date`
* **Nilai (Expression)**:
  ```javascript
  {{ $now.toFormat('dd LLLL yyyy') }}
  ```
* **Keterangan**: Menghasilkan tanggal hari ini secara otomatis (Contoh: `17 Juli 2026`).

### **2. Field 2: `period`**
* **Tipe Data**: `String`
* **Nama Field**: `period`
* **Nilai (Expression)**:
  * **Opsi A (Jika laporan pemantauan 2 jam ke belakang)**:
    ```javascript
    {{ $now.minus({ hours: 2 }).toFormat('HH:mm') }} - {{ $now.toFormat('HH:mm') }} WITA
    ```
    *(Contoh hasil: `15:20 - 17:20 WITA`)*
  * **Opsi B (Jika laporan bulanan)**:
    ```javascript
    {{ $now.minus({ months: 1 }).toFormat('LLLL yyyy') }}
    ```
    *(Contoh hasil: `Juni 2026`)*

### **3. Field 3: `server`**
* **Tipe Data**: `String`
* **Nama Field**: `server`
* **Nilai (Statis)**:
  ```text
  ilhamvm2
  ```
* **Keterangan**: Nama Host/VM tempat server Anda berada.

### **4. Field 4: `reported_by`**
* **Tipe Data**: `String`
* **Nama Field**: `reported_by`
* **Nilai (Statis)**:
  ```text
  n8n Server Automation
  ```
* **Keterangan**: Sistem otomatisasi yang merender laporan ini.

### **5. Field 5: `pic`**
* **Tipe Data**: `String`
* **Nama Field**: `pic`
* **Nilai (Statis)**:
  ```text
  Ilham
  ```
* **Keterangan**: Nama penanggung jawab server yang menerima/mengelola VM.

---

## 3. Hasil Output JSON di n8n

Setelah node ini dijalankan, n8n akan menghasilkan struktur JSON bersih seperti berikut:

```json
{
  "report_date": "17 Juli 2026",
  "period": "15:20 - 17:20 WITA",
  "server": "ilhamvm2",
  "reported_by": "n8n Server Automation",
  "pic": "Ilham"
}
```

Data JSON ini kemudian langsung dihubungkan ke properti halaman Notion pada langkah berikutnya.
