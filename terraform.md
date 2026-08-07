# Dokumentasi Penggunaan Terraform di VPS

Dokumen ini berisi penjelasan detail mengenai tujuan penggunaan Terraform, langkah-langkah instalasi yang rinci, serta penjelasan mengenai file-file yang dibuat dalam siklus hidup proyek Terraform.

---

## 🎯 1. Tujuan Penggunaan Terraform

Terraform adalah alat **Infrastructure as Code (IaC)** yang dikembangkan oleh HashiCorp. Tujuan utama penggunaannya di VPS adalah:

1. **Otomatisasi Penuh (Automation)**:  
   Menghilangkan kebutuhan untuk membuat, mengonfigurasi, dan menghubungkan kontainer Docker secara manual lewat terminal. Semua didefinisikan dalam bentuk file kode `.tf`.
   
2. **State Management & Konsistensi**:  
   Terraform melacak semua infrastruktur yang telah dibuat melalui file `terraform.tfstate`. Ini memastikan konfigurasi server Anda tidak bergeser (*drift detection*) dan memudahkan pembersihan sistem secara bersih menggunakan satu perintah (`terraform destroy`).

3. **Portabilitas & Skalabilitas (Reproducibility)**:  
   Mempermudah penggandaan atau pemindahan sistem. Kode infrastruktur yang sama dapat di-copy ke VPS lain dan dideploy instan dalam hitungan detik.

4. **Integrasi GitOps**:  
   Memungkinkan penggabungan dengan CI/CD tools seperti **Jenkins**, sehingga perubahan server dapat dipicu otomatis saat ada perubahan kode yang di-push ke Git repository.

---

## 🛠️ 2. Langkah Detail Instalasi di VPS (Ubuntu/Debian)

Berikut adalah panduan instalasi mendalam beserta penjelasan fungsi dari setiap perintah yang dijalankan.

### A. Install Docker (Prasyarat Utama)
Terraform memerlukan "provider" untuk mengeksekusi perintah. Dalam latihan ini, kita menggunakan Docker sebagai penyedia infrastruktur kontainer di VPS.

```bash
# 1. Update daftar paket sistem Anda agar mendapatkan versi terbaru
sudo apt-get update

# 2. Unduh dan install aplikasi Docker Engine di VPS
sudo apt-get install -y docker.io

# 3. Jalankan service Docker dan atur agar otomatis aktif saat VPS dinyalakan (booting)
sudo systemctl start docker
sudo systemctl enable docker

# 4. Daftarkan user sistem Anda (misal: ilham) ke dalam grup 'docker'
# Tujuannya agar Anda bisa menjalankan perintah 'docker' tanpa perlu menuliskan 'sudo'
sudo usermod -aG docker $USER
```
> [!IMPORTANT]  
> Setelah menjalankan perintah `usermod`, Anda **harus keluar (log out) dari sesi SSH** dan masuk kembali agar hak akses Docker yang baru diterapkan pada user Anda.

---

### B. Install Terraform
Instalasi dilakukan dengan menambahkan repository resmi milik HashiCorp agar Anda selalu mendapatkan update versi yang aman dan stabil.

```bash
# 1. Install paket gnupg dan software-properties-common
# Ini diperlukan untuk mengelola kunci keamanan (GPG) dari repository eksternal
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

# 2. Unduh kunci keamanan (GPG Key) resmi dari HashiCorp dan simpan ke sistem
# Kunci ini memastikan bahwa paket Terraform yang kita download adalah asli dan belum dimodifikasi
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

# 3. Tambahkan alamat repository HashiCorp ke daftar sumber paket VPS Anda
# Perintah '$(lsb_release -cs)' otomatis mendeteksi versi Ubuntu Anda (misal: focal, jammy, noble)
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# 4. Update ulang daftar paket untuk mengenali repository baru, lalu pasang Terraform
sudo apt-get update && sudo apt-get install -y terraform
```

---

### C. Verifikasi Instalasi
Pastikan Terraform sudah terpasang dengan menjalankan perintah:
```bash
terraform -version
```
Jika muncul keterangan versi (seperti `Terraform v1.x.x`), maka instalasi selesai dan siap digunakan.

---

## 📁 3. Daftar File yang Dibuat dalam Proyek Terraform

Ketika membuat proyek Terraform, terdapat dua jenis file: **file yang Anda buat secara manual** (konfigurasi), dan **file yang dibuat otomatis oleh Terraform** setelah perintah dijalankan.

### A. File yang Dibuat Secara Manual (Konfigurasi Anda)

| Nama File | Fungsi |
| :--- | :--- |
| **`main.tf`** | File utama tempat Anda mendeklarasikan provider (misal: Docker) dan semua resource infrastruktur yang ingin dibangun (seperti container, network, dan volume). |
| **`variables.tf`** | File opsional untuk mendefinisikan variabel/parameter input. Berguna agar konfigurasi di `main.tf` bersifat dinamis (tidak di-*hardcode*). |
| **`outputs.tf`** | File opsional untuk menentukan data apa saja yang ingin dicetak ke layar setelah instalasi berhasil (misal: mencetak IP Address server atau URL web). |

---

### B. File yang Dibuat Otomatis oleh Terraform (Siklus Hidup Sistem)

File-file di bawah ini akan muncul otomatis di dalam folder proyek Anda setelah Anda menjalankan perintah Terraform:

```text
📂 folder-proyek/
├── 📂 .terraform/            (Otomatis - Direktori berisi file binary plugin)
├── 📄 .terraform.lock.hcl    (Otomatis - Mengunci versi plugin)
├── 📄 terraform.tfstate      (Otomatis - Database status infrastruktur nyata)
└── 📄 terraform.tfstate.backup (Otomatis - Cadangan database status sebelumnya)
```

#### 1. Folder `.terraform/` (Direktori Tersembunyi)
* **Kapan muncul**: Setelah menjalankan perintah `terraform init`.
* **Fungsi**: Berisi file-file binary (plugin) provider yang telah diunduh. Karena kita menggunakan provider Docker di `main.tf`, folder ini akan berisi driver khusus agar Terraform dapat memerintahkan Docker Daemon di VPS.

#### 2. File `.terraform.lock.hcl`
* **Kapan muncul**: Setelah menjalankan perintah `terraform init`.
* **Fungsi**: Mengunci versi provider yang diunduh. Hal ini menjamin bahwa jika orang lain (atau pipeline Jenkins) menjalankan kode Anda di komputer lain, mereka akan menggunakan versi driver provider yang sama persis untuk mencegah ketidakcocokan (*compatibility issue*).

#### 3. File `terraform.tfstate` (Sangat Penting)
* **Kapan muncul**: Setelah menjalankan perintah `terraform apply` pertama kali.
* **Fungsi**: Berisi database format JSON yang menyimpan status nyata dari infrastruktur yang telah dideploy di VPS (seperti ID container, IP internal, dan port).
* **Aturan Penting**: **Jangan pernah mengedit atau menghapus file ini secara manual!** Jika file ini hilang, Terraform akan kehilangan ingatan tentang server apa saja yang sudah dia buat.

#### 4. File `terraform.tfstate.backup`
* **Kapan muncul**: Muncul setelah Anda melakukan perubahan konfigurasi dan menjalankan `terraform apply` untuk kedua kalinya.
* **Fungsi**: Berisi salinan cadangan otomatis dari file `terraform.tfstate` versi sebelumnya sebelum perubahan terbaru diterapkan. Ini digunakan sebagai penyelamat jika proses apply terbaru mengalami kegagalan di tengah jalan.
