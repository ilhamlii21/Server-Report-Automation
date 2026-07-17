# Panduan Instalasi n8n Self-Hosted dengan Node.js & PM2

Dokumen ini berisi panduan lengkap untuk memasang (install) n8n secara native di VM/Server menggunakan Node.js dan mengelolanya menggunakan process manager PM2 agar tetap berjalan di latar belakang (background).

---

## 1. Persiapan: Instalasi Node.js (v20)

n8n memerlukan runtime Node.js. Jalankan perintah berikut untuk menginstal Node.js versi LTS terbaru:

```bash
# Menambahkan repository NodeSource untuk Node.js v20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

# Menginstal Node.js dan npm
sudo apt-get install -y nodejs
```

Verifikasi hasil instalasi:
```bash
node -v
npm -v
```

---

## 2. Instalasi n8n secara Global

Gunakan `npm` untuk mengunduh dan menginstal n8n secara global di sistem:

```bash
sudo npm install n8n -g
```

Verifikasi instalasi n8n:
```bash
n8n -v
```

---

## 3. Instalasi dan Konfigurasi PM2 (Process Manager)

PM2 digunakan untuk menjaga agar proses n8n tetap berjalan secara terus-menerus di latar belakang meskipun sesi SSH Anda terputus.

1. **Install PM2 secara global:**
   ```bash
   sudo npm install pm2 -g
   ```

2. **Jalankan n8n pertama kali menggunakan PM2:**
   Dengan konfigurasi khusus untuk menonaktifkan keamanan cookie HTTPS (agar bisa diakses menggunakan HTTP biasa) dan mengizinkan node **Execute Command** berjalan:
   ```bash
   NODES_EXCLUDE=[] N8N_SECURE_COOKIE=false pm2 start n8n
   ```

3. **Konfigurasikan Auto-Start saat VM Booting:**
   ```bash
   pm2 startup
   ```
   *Perintah di atas akan menghasilkan sebuah perintah konfigurasi sistem otomatis (dimulai dengan `sudo env...`). Salin (copy) perintah tersebut dari terminal Anda, tempelkan (paste), lalu jalankan.*

4. **Simpan konfigurasi proses PM2:**
   ```bash
   pm2 save
   ```

---

## 4. Perintah Pengelolaan n8n via PM2

Berikut adalah perintah-perintah CLI yang sering digunakan untuk mengelola n8n:

* **Melihat status aplikasi:**
  ```bash
  pm2 status
  ```
* **Melihat log/output real-time n8n:**
  ```bash
  pm2 logs n8n
  ```
* **Me-restart n8n (sambil memperbarui environment variable):**
  ```bash
  NODES_EXCLUDE=[] N8N_SECURE_COOKIE=false pm2 restart n8n --update-env
  ```
* **Menghentikan n8n:**
  ```bash
  pm2 stop n8n
  ```

---

## 5. Mengakses Halaman Web GUI n8n

Setelah n8n berjalan, Anda dapat mengakses dashboard GUI interaktif melalui browser laptop Anda:

👉 **`http://<IP_PUBLIC_VM_ANDA>:5678`**

### ⚠️ Masalah Umum: Halaman Tidak Bisa Diakses (Loading Terus)
Jika Anda menggunakan cloud provider seperti **Azure**, Anda harus membuka akses port masuk (**Inbound Port Rule**) pada **Network Security Group (NSG)** VM Anda untuk port **`5678`** agar koneksi dari browser Anda tidak diblokir.
