# Panduan Lengkap: Pemisahan Jenkins Controller & Runner ke VM Berbeda dan Pengujian Metrik Real-time

Dokumen ini berisi panduan teknis langkah demi langkah untuk memisahkan **Jenkins Controller** dan **Jenkins Runner (Agent)** ke VM yang berbeda, serta skenario pengujian menggunakan Prometheus dan Grafana untuk melihat lonjakan (*resource spike*) secara nyata.

---

## 1. Konsep Dasar: Controller vs Runner

*   **Jenkins Controller (VM 1):** Bertindak sebagai "Otak". Mengelola UI, konfigurasi, kredensial, dan penjadwalan. Beban kerja ringan ($\approx$ 10-15%).
*   **Jenkins Runner/Agent (VM 2):** Bertindak sebagai "Otot". Mengeksekusi *compile*, *testing*, *bundling*, dan deployment. Beban kerja berat ($\approx$ 85-90%).

---

## 2. Langkah-Langkah Pemisahan Runner ke VM 2

### **Langkah A: Persiapan di VM 2 (Runner)**
Masuk ke terminal VM 2 dan instal Java Runtime (JRE) karena agen Jenkins dijalankan dengan Java:

```bash
# 1. Update package manager
sudo apt update

# 2. Instal Java JRE 17 (Sesuaikan dengan versi Java di Jenkins Controller Anda)
sudo apt install openjdk-17-jr-headless -y

# 3. Buat direktori kerja khusus untuk agen Jenkins
mkdir -p /home/ilhamvm2/jenkins-agent
```

### **Langkah B: Daftarkan Kredensial SSH VM 2 di Jenkins**
1. Buka dashboard Jenkins Anda (di VM 1).
2. Masuk ke **Manage Jenkins** -> **Credentials** -> **System** -> **Global credentials**.
3. Klik **Add Credentials**:
   *   **Kind:** `SSH Username with private key`
   *   **ID:** `vm2-runner-key`
   *   **Username:** `ilhamvm2`
   *   **Private Key:** Pilih *Enter directly* -> klik *Add* -> tempelkan (*paste*) isi kunci privat SSH Anda (`~/.ssh/id_rsa` dari VM 1).
   *   Klik **Create**.

### **Langkah C: Daftarkan Node Agent Baru**
1. Buka **Manage Jenkins** -> **Nodes** -> Klik **New Node** di sisi kiri.
2. Isi detail:
   *   **Node name:** `VM2-Runner`
   *   **Type:** Pilih `Permanent Agent`
   *   Klik **Create**.
3. Lengkapi konfigurasi sebagai berikut:
   *   **Description:** `Runner untuk testing dan deploy di VM 2`
   *   **Number of executors:** `2`
   *   **Remote root directory:** `/home/ilhamvm2/jenkins-agent`
   *   **Labels:** `runner-vm2` *(ini label pemanggil di Jenkinsfile)*
   *   **Usage:** `Use this node as much as possible`
   *   **Launch method:** Pilih **Launch agents via SSH**
       *   **Host:** Isi dengan IP VM 2 Anda (`20.205.17.18` atau `20.205.24.157`)
       *   **Credentials:** Pilih `vm2-runner-key`
       *   **Host Key Verification Strategy:** Pilih **Non-verifying Verification Strategy**
4. Klik **Save**.
5. Buka node `VM2-Runner` tersebut dan periksa **Log**. Jika sukses, status akan berubah menjadi **Active/Online**.

---

## 3. Uji Coba Simulasi Beban & Monitoring dengan Prometheus/Grafana

Untuk membuktikan perbedaan beban kerja secara riil, kita akan menjalankan skenario simulasi *high-CPU* menggunakan tool Linux bernama `stress`.

### **Langkah A: Instalasi Tool `stress` di VM Target**
Instal tool stress generator di VM 1 dan VM 2 agar siap dieksekusi oleh pipeline:
```bash
sudo apt install stress -y
```

### **Langkah B: Jalankan Skenario 1 (Single VM - Digabung)**
Buat pipeline dengan agen default (`agent any`) yang berjalan di VM 1 (Controller):

```groovy
pipeline {
    agent any // Berjalan di VM 1 (Controller)
    stages {
        stage('Simulasi Compile Berat') {
            steps {
                echo 'Memulai kompilasi berat di VM 1...'
                sh 'stress --cpu 2 --vm 1 --vm-bytes 512M --timeout 90s'
            }
        }
    }
}
```
**Hasil pada Grafana:**
*   Grafik CPU VM 1 (Controller) akan langsung melonjak hingga **100%**.
*   Dashboard UI Jenkins akan menjadi lambat diakses (*lag/freeze*) karena kehabisan resource CPU.

---

### **Langkah C: Jalankan Skenario 2 (Distributed VM - Dipisah)**
Ganti deklarasi agen untuk menggunakan Runner baru di VM 2:

```groovy
pipeline {
    agent {
        label 'runner-vm2' // Berjalan di VM 2 (Runner)
    }
    stages {
        stage('Simulasi Compile Berat') {
            steps {
                echo 'Memulai kompilasi berat di VM 2...'
                sh 'stress --cpu 2 --vm 1 --vm-bytes 512M --timeout 90s'
            }
        }
    }
}
```
**Hasil pada Grafana:**
*   **Grafik CPU VM 1 (Controller):** Tetap datar dan stabil di kisaran **10-15%** (dashboard Jenkins tetap responsif dan lancar dibuka).
*   **Grafik CPU VM 2 (Runner):** Mengalami lonjakan (*spike*) tajam hingga **100%** selama 90 detik, membuktikan seluruh beban kerja kompilasi sukses didelegasikan ke VM 2.
