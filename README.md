# Mobile Proxy Manager (Production-Ready)

Sistem lengkap, stabil, aman, dan siap produksi untuk membuat dan mengelola mobile proxy berbasis SIM Card / 4G / 5G atau WiFi dari perangkat Android / Termux milik sendiri.

---

## 1. Arsitektur Sistem

Sistem ini memisahkan secara ketat antara **Control Plane** (VPS) dan **Data Plane** (Android / Termux).

```
========================================================================
                      CONTROL PLANE (VPS)
========================================================================
  [ Admin Browser ]
         │ (HTTP / HTTPS / SSE)
         ▼
  [ Nginx :80/:443 ]
         │
         ├─────────────────────────────────────────┐
         ▼                                         ▼
  [ Frontend SPA ]                          [ Backend Node.js ]
                                                   │
         ┌──────────────────┬──────────────────────┼──────────────────┐
         ▼                  ▼                      ▼                  ▼
    [ SQLite DB ]    [ Agent Manager ]     [ Health Checker ]  [ Rotation Manager ]
                            ▲
                            │ (HTTPS / SSE Authenticated)
============================│===========================================
                      DATA PLANE (ANDROID / TERMUX)
============================│===========================================
                            │
                     [ Termux Agent ]
                            │ (Local Control)
                            ▼
                  [ SOCKS5 Proxy Server ]
                            │
                            ▼
                   [ Android Network ]
                            │
                            ▼
               [ SIM Card 4G/5G  ATAU  WiFi ]
                            │
                            ▼
                      [ Internet ]
                            │
                            ▼
                   [ Proxy Egress IP ]
```

### Mode Koneksi Proxy:
1. **Mode A — Direct Mobile Proxy**:
   Digunakan jika jaringan seluler atau Android memberikan alamat IP publik langsung yang dapat menerima traffic masuk (misal port forwarding / IPv6 seluler / WiFi lokal).
2. **Mode B — VPS Relay / Reverse Tunnel (Paling Umum & Stabil)**:
   Karena hampir semua operator seluler (Telkomsel, Indosat, XL, Tri, Smartfren) maupun WiFi ISP berada di balik **Carrier-Grade NAT (CGNAT)** dan tidak bisa menerima koneksi masuk secara langsung, Termux membuat reverse tunnel ke VPS. Client terhubung ke port VPS, lalu traffic diteruskan ke Termux melalui reverse tunnel, dan **traffic keluar (egress) ke Internet tetap 100% melalui SIM Card / WiFi Android Anda**.

> **PENTING TENTANG DATA VALIDITY & REAL-TIME USAGE**:
> - Public IP VPS **bukanlah** IP proxy. Egress IP selalu diverifikasi langsung melalui koneksi proxy itu sendiri. Jika hasil IP tidak dapat diverifikasi atau terjadi IP conflict, status ditandai `UNVERIFIED` atau `IP_CONFLICT`.
> - Penggunaan kuota / data proxy dihitung langsung pada socket level di SOCKS5 Agent (Termux) secara presisi (`bytes_in` / download & `bytes_out` / upload). Statistik ditransmisikan via SSE (Server-Sent Events) ke Web Dashboard secara real-time.

---

## 2. PANDUAN SETUP DARI AWAL SAMPAI PROXY AKTIF

Ikuti 5 langkah berurutan di bawah ini untuk mengaktifkan sistem dari awal:

### LANGKAH 1: Setup VPS (Server Control Plane)

1. Masuk ke VPS Anda via SSH:
   ```bash
   ssh -i key.pem ubuntu@IP_VPS_ANDA
   ```
2. Clone repository atau masuk ke folder `proxy-manager`:
   ```bash
   cd /home/ubuntu/proxy-manager
   ```
3. Jalankan script installer VPS:
   ```bash
   sudo ./install.sh
   ```
   *(Atau jalankan non-interaktif: `sudo ./install.sh --user admin --password "PasswordKuat123" -y`)*
4. Setelah selesai, installer akan menampilkan:
   - **Dashboard URL**: `http://IP_VPS_ANDA/`
   - **Admin Username**: `admin`
   - **Admin Password**: Password yang Anda buat
5. Buka browser di komputer Anda dan akses `http://IP_VPS_ANDA/`. Login dengan akun admin.

---

### LANGKAH 2: Hubungkan HP Android / Termux (Data Plane)

1. Buka **Web Dashboard** di browser, klik menu **📱 Termux Agents**, lalu klik tombol **+ Pair New Agent**.
2. Dashboard akan menampilkan dialog berisi Token Pairing (berlaku 15 menit).
3. Buka aplikasi **Termux** di HP Android Anda.
4. Jalankan perintah auto-install di Termux (pilih sesuai koneksi HP Anda):

- **Opsi A: Menggunakan Data Internet Seluler (SIM Card 4G/5G)**:
  ```bash
  curl -fsSL http://IP_VPS_ANDA/install-agent.sh | bash -s -- http://IP_VPS_ANDA TOKEN_DARI_DASHBOARD "HP-Seluler" cellular
  ```
- **Opsi B: Menggunakan WiFi (WiFi Rumah / CBN / Kantor)**:
  ```bash
  curl -fsSL http://IP_VPS_ANDA/install-agent.sh | bash -s -- http://IP_VPS_ANDA TOKEN_DARI_DASHBOARD "HP-WiFi" wifi
  ```
- **Opsi C: Mode Interaktif (Pilih Menu di Layar HP)**:
  ```bash
  curl -fsSL http://IP_VPS_ANDA/install-agent.sh | bash
  ```

5. Script akan otomatis:
   - Memasang dependensi (Python, OpenSSH, Autossh, Curl, Tar).
   - Mengaktifkan `termux-wake-lock` agar proses tidak mati saat layar HP terkunci.
   - Melakukan pairing dengan VPS secara aman.
   - **Langsung berjalan di background** bersama pengawas auto-restart (`proxy-supervisor.sh`).
   - Memasang hook **Termux:Boot** agar otomatis menyala kembali saat HP di-reboot.

---

### LANGKAH 3: Buat Proxy di Web Dashboard

1. Di Web Dashboard, klik menu **🌐 Proxies**, lalu klik tombol **+ Add New Proxy**.
2. Isi form:
   - **Proxy Name**: Misal `Mobile-Proxy-01`
   - **Listen Port**: Misal `10001` *(port dibuka di firewall VPS)*
   - **Bind to Termux Agent**: **PENTING: Pilih perangkat HP Android Anda dari dropdown!**
   - **Proxy Username**: Misal `userproxy1`
   - **Proxy Password**: Password untuk proxy client (minimal 8 karakter, misal `PassRahasia123`).
3. Klik **Create Proxy**.

---

### LANGKAH 4: Aktifkan Tunnel Relay di Termux

Buka kembali Termux di HP Android Anda dan jalankan perintah tunnel:

```bash
~/start-tunnel.sh 16.79.169.71 22 ubuntu 10001 10001
```
*(Atau cukup ketik `~/start-tunnel` jika tanpa parameter).*

Cek statusnya di Termux:
```bash
~/status-agent.sh
```
Pastikan tertulis:
- `[ONLINE] Termux Agent is RUNNING (Background).`
- `[ACTIVE] Auto-Restart Watchdog is GUARDING.`
- `[TUNNEL] Reverse Tunnel is ACTIVE.`

---

### LANGKAH 5: Verifikasi Status & Mulai Gunakan Proxy

1. Di Web Dashboard ([http://IP_VPS_ANDA/](http://IP_VPS_ANDA/)), masuk ke menu **Proxies**.
2. Klik tombol **⚡ Test** pada proxy yang baru Anda buat.
3. Status proxy akan berubah menjadi **ONLINE**, dan Public IP serta Latensi akan muncul!
4. Proxy sekarang aktif 100% dan siap digunakan dengan detail:
   - **Host / IP**: `IP_VPS_ANDA`
   - **Port**: `10001`
   - **Type**: `SOCKS5`
   - **Username**: `userproxy1`
   - **Password**: `PassRahasia123`

---

## 3. CARA MEMATIKAN PROXY (STOP PROXY)

Anda dapat mematikan proxy melalui 3 cara:

### Cara 1: Mematikan dari HP Android (Termux)
- **Hanya Mematikan Tunnel (Menutup akses publik ke proxy)**:
  ```bash
  ~/stop-tunnel.sh
  ```
  *(Atau: `~/stop-tunnel`)*
  *Port 10001 di VPS akan langsung tertutup dan tidak bisa diakses dari luar, sementara Agent tetap melapor ke dashboard.*

- **Mematikan Seluruh Service (Agent, Watchdog, dan Tunnel)**:
  ```bash
  ~/stop-agent.sh
  ```
  *(Atau: `~/stop-agent`)*
  *Semua proses SOCKS5 server, agent python, auto-restart watchdog, dan tunnel akan dihentikan total dan wake-lock dilepas.*

---

### Cara 2: Mematikan dari Web Dashboard
1. Buka Web Dashboard di browser -> Masuk ke menu **Proxies**.
2. Klik tombol **Delete (🗑️)** jika ingin menghapus proxy, atau hapus binding agent.
3. VPS akan otomatis mengirimkan sinyal `STOP` ke Agent di HP Android.

---

### Cara 3: Mematikan dari Terminal VPS (CLI)
Jika Anda sedang membuka SSH VPS:
- Hentikan backend service VPS:
  ```bash
  proxy-manager stop
  ```
- Atau tutup port firewall tertentu:
  ```bash
  sudo ufw delete allow 10001/tcp
  ```

---

## 4. CARA MENGAKTIFKAN KEMBALI PROXY (START / RE-ACTIVATE)

Jika sebelumnya proxy dimatikan dan Anda ingin menyalakannya kembali:

### Langkah 1: Di HP Android (Termux)
Buka aplikasi Termux dan jalankan:
1. **Nyalakan Agent & Watchdog**:
   ```bash
   ~/start-agent.sh
   ```
   *(Atau: `~/start-agent`)*
2. **Nyalakan Tunnel**:
   ```bash
   ~/start-tunnel.sh 16.79.169.71 22 ubuntu 10001 10001
   ```
   *(Atau: `~/start-tunnel`)*
3. **Pastikan Status Berjalan**:
   ```bash
   ~/status-agent.sh
   ```

### Langkah 2: Di Web Dashboard
1. Buka Web Dashboard di browser -> Masuk ke menu **Proxies**.
2. Klik tombol **⚡ Test** pada proxy Anda.
3. Dashboard akan langsung menguji TCP handshake, otentikasi SOCKS5, dan mendeteksi IP keluar seluler. Status akan langsung kembali **ONLINE**!

---

## 5. CARA MENGGUNAKAN PROXY DI CLIENT / APLIKASI

### A. Uji Menggunakan cURL (Terminal / CMD)
```bash
curl -x socks5h://userproxy1:PassRahasia123@IP_VPS_ANDA:10001 https://api.ipify.org
```
*(Gunakan `socks5h://` agar DNS direlay langsung melalui koneksi HP Android).*

### B. Menggunakan di Browser (Chrome / Firefox / Brave)
Gunakan ekstensi **FoxyProxy Standard** atau **Proxy SwitchyOmega**:
1. Pasang ekstensi FoxyProxy di browser.
2. Tambahkan proxy baru:
   - **Type**: SOCKS5
   - **Server**: `IP_VPS_ANDA`
   - **Port**: `10001`
   - **Username**: `userproxy1`
   - **Password**: `PassRahasia123`
   - Centang opsi: **Send DNS through SOCKS5 proxy**.
3. Buka situs [https://whoer.net](https://whoer.net) atau [https://ipinfo.io](https://ipinfo.io) untuk melihat IP publik Anda.

### C. Menggunakan di Script Python (Requests)
```python
import requests

proxies = {
    'http': 'socks5h://userproxy1:PassRahasia123@IP_VPS_ANDA:10001',
    'https': 'socks5h://userproxy1:PassRahasia123@IP_VPS_ANDA:10001',
}

resp = requests.get('https://api.ipify.org?format=json', proxies=proxies, timeout=10)
print("Proxy Egress IP:", resp.json()['ip'])
```

---

## 6. CARA MEROTASI IP (GANTI IP SELULER)

1. **Rotasi Manual via Web Dashboard**:
   - Buka menu **Proxies** di Web Dashboard -> Klik tombol **🔄 Rotate**.
   - Agent di HP akan memicu reset jaringan seluler / mode pesawat.
   - Dalam 5-10 detik, IP baru akan terdeteksi dan tercatat di menu **IP History**.
2. **Rotasi Terjadwal Otomatis**:
   - Buka menu **Settings** di Web Dashboard.
   - Atur `Default Rotation Interval (seconds)` misalnya `600` (setiap 10 menit).
   - Sistem akan merotasi IP secara berkala tanpa perlu intervensi manual.

---

## 7. TIPS STABILITAS ANDROID (PENTING)

Agar Termux dapat berjalan 24 jam nonstop tanpa dimatikan Android:
1. **Matikan Penghemat Baterai (Battery Optimization)**:
   - Buka *Pengaturan HP -> Aplikasi -> Termux -> Baterai -> Pilih "Tidak Dibatasi" (Unrestricted / Don't optimize)*.
2. **Kunci Aplikasi di Recent Apps**:
   - Buka Recent Apps di HP Anda, tahan jendela Termux, lalu pilih ikon **Gembok (Lock)** agar tidak terhapus saat membersihkan RAM.
3. **Pastikan Wake Lock Aktif**:
   - Script `start-agent.sh` sudah otomatis memanggil `termux-wake-lock`.

---

## 8. PERINTAH LENGKAP CLI VPS

Kelola VPS dari terminal menggunakan perintah `proxy-manager`:

```bash
# Cek Diagnostik & Status
proxy-manager test
proxy-manager status

# Kontrol Service Backend
proxy-manager restart
proxy-manager stop
proxy-manager start

# Melihat Data
proxy-manager proxy list
proxy-manager agent list
proxy-manager ip
proxy-manager logs

# Backup & Restore
proxy-manager backup
proxy-manager restore backups/backup_TIMESTAMP.tar.gz

# Uninstall
proxy-manager uninstall
```
