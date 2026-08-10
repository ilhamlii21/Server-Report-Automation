# Panduan Simulasi Lonjakan (Spike) RAM pada Jenkins Runner Menggunakan `stress`

Dokumen ini menjelaskan cara mensimulasikan penggunaan memori (RAM) yang tinggi pada **Jenkins Runner (VM 2)** untuk kebutuhan pengujian metrik real-time di Prometheus dan Grafana. 

Tujuan simulasi ini adalah untuk membuktikan secara visual bahwa beban memori yang berat dari proses *build* dapat diisolasi sepenuhnya di VM Runner tanpa membebani *Jenkins Controller* (VM 1).

---

## 1. Prasyarat

Pastikan tool **`stress`** sudah terpasang di VM 2 (Runner). Jika belum, jalankan perintah berikut di terminal VM 2:
```bash
sudo apt update && sudo apt install stress -y
```

---

## 2. Konfigurasi `Jenkinsfile`

Tambahkan tahapan (*stage*) simulasi ini di dalam berkas `Jenkinsfile` Anda, tepat di bawah tahapan *Checkout* dan sebelum tahapan *Deploy*:

```groovy
        // Tahap 1: Mendeteksi branch yang aktif
        stage('Checkout') {
            steps {
                echo "Mendeteksi branch yang aktif: ${env.BRANCH_NAME}"
            }
        }

        // --- STAGE SIMULASI RAM SPIKE ---
        stage('Simulasi Spike RAM di VM 2') {
            when {
                branch 'staging' // Hanya berjalan di branch staging
            }
            steps {
                echo 'Memulai alokasi RAM sebesar 1.00 GiB di VM 2 selama 60 detik...'
                
                // Menjalankan simulasi stress memori virtual (RAM)
                sh 'stress --vm 1 --vm-bytes 1G --vm-hang 60 --timeout 60s'
            }
        }
```

---

## 3. Penjelasan Parameter Perintah `stress`

| Parameter | Fungsi |
| :--- | :--- |
| `--vm 1` | Menjalankan 1 proses pekerja (*worker*) untuk alokasi memori virtual. |
| `--vm-bytes 1G` | Jumlah kapasitas RAM yang ingin dialokasikan (misalnya `1G` untuk 1 GiB, `1.5G` untuk 1.5 GiB). |
| `--vm-hang 60` | Menginstruksikan program untuk **menahan (hang)** memori yang sudah dialokasikan selama 60 detik sebelum dilepaskan kembali. Opsi ini krusial agar grafik di Grafana terlihat naik dan bertahan rata di atas. |
| `--timeout 60s` | Batas waktu durasi eksekusi program (60 detik) untuk menghindari program berjalan selamanya secara tidak sengaja. |

*Catatan: Kami sengaja tidak menggunakan opsi `--cpu` agar penggunaan inti CPU tidak melonjak ke 100% dan fokus lonjakan hanya terjadi di grafik RAM.*

---

## 4. Hasil Visualisasi Grafana

Setelah pipeline Jenkins dijalankan, Prometheus dan Grafana akan menangkap data aktivitas sistem sebagai berikut:

*   **Grafik RAM VM 2 (Runner):**
    Akan terlihat grafik yang melonjak tajam ke angka **`1.00 GiB`** (berlabel `ilhamvm2:stress`) tepat pada waktu eksekusi pipeline dimulai, bertahan mendatar selama 60 detik, lalu langsung turun drastis kembali ke kondisi semula setelah tugas selesai.
*   **Grafik RAM VM 1 (Controller):**
    Penggunaan RAM di VM 1 akan tetap stabil, datar, dan aman tanpa ada lonjakan memori sama sekali, menandakan dashboard Jenkins Anda aman dari risiko *crash* akibat kehabisan memori (*Out of Memory*).
