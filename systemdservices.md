# Panduan Konfigurasi Pelacakan Service Systemd dengan Prometheus (Docker)

Dokumen ini menjelaskan langkah-langkah untuk mengaktifkan **systemd-collector** pada `node-exporter` yang berjalan di dalam Docker, guna memantau status keaktifan layanan systemd host (seperti NGINX, Docker, MySQL) secara langsung menggunakan Prometheus dan n8n.

---

## 1. Pendahuluan

Secara default, kontainer `node-exporter` tidak dapat membaca status service systemd pada host VM karena keterbatasan isolasi Docker dan pembatasan keamanan **AppArmor**. 

Dengan melakukan mounting socket D-Bus host dan menonaktifkan profil AppArmor secara khusus untuk kontainer ini, `node-exporter` dapat berkomunikasi dengan systemd host untuk menghasilkan metrik status unit.

---

## 2. Langkah 1: Konfigurasi `docker-compose.yml`

Buka file konfigurasi docker-compose Anda di VM:
```bash
nano ~/monitoring/docker-compose.yml
```

Perbarui bagian service `node-exporter` dengan menambahkan konfigurasi `security_opt`, volume mount `dbus`, serta flag `--collector.systemd` pada `command`:

```yaml
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    # 1. Nonaktifkan batasan AppArmor untuk akses D-Bus host
    security_opt:
      - apparmor:unconfined
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
      # 2. Mount socket D-Bus systemd milik host VM
      - /var/run/dbus/system_bus_socket:/var/run/dbus/system_bus_socket:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/host/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
      # 3. Aktifkan kolektor metrik systemd
      - '--collector.systemd'
    restart: unless-stopped
```

---

## 3. Langkah 2: Mengapa Memerlukan `apparmor:unconfined`?

Pada beberapa sistem operasi (seperti Ubuntu Azure), default profile AppArmor Docker (`docker-default`) akan melarang kontainer mengirimkan pesan D-Bus ke host VM meskipun socket-nya sudah di-mount. Tanpa konfigurasi ini, log kontainer akan memunculkan error:
> *`couldn't get dbus connection: An AppArmor policy prevents this sender from sending this message...`*

Menggunakan `apparmor:unconfined` memberikan izin yang cukup bagi `node-exporter` untuk membaca status sistem layanan secara aman.

---

## 4. Langkah 3: Terapkan Perubahan

Terapkan pembaruan konfigurasi tersebut dengan mematikan dan menghidupkan ulang kontainer menggunakan Docker Compose di dalam folder `~/monitoring`:

```bash
cd ~/monitoring
docker compose down && docker compose up -d
```

---

## 5. Langkah 4: Pengujian & Integrasi ke n8n

Untuk memverifikasi metrik, buka Prometheus Web UI (`http://<IP_VM>:9090`) dan cari metrik berikut:
```promql
node_systemd_unit_state
```

### **Implementasi Query pada n8n (HTTP Request):**

Untuk memeriksa status keaktifan service tertentu pada laporan otomatis Anda, gunakan HTTP Request Node dengan query parameter berikut:

* **Mengecek Status NGINX:**
  ```promql
  node_systemd_unit_state{name="nginx.service", state="active"}
  ```
* **Mengecek Status Docker:**
  ```promql
  node_systemd_unit_state{name="docker.service", state="active"}
  ```

**Arti Nilai Output (Value):**
* **`1`** = Service dalam kondisi **Aktif / Running** (Hijau)
* **`0`** = Service dalam kondisi **Mati / Down** (Merah)
