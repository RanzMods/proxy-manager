# Mobile Proxy Manager (Production-Ready)

Sistem lengkap, stabil, aman, dan siap produksi untuk membuat dan mengelola mobile proxy berbasis SIM Card / 4G / 5G dari perangkat Android / Termux milik sendiri.

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
                    [ SIM Card / LTE ]
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
2. **Mode B — VPS Relay / Reverse Tunnel (Paling Umum untuk Seluler)**:
   Karena hampir semua operator seluler (Telkomsel, Indosat, XL, Tri, Smartfren) menempatkan perangkat di belakang **Carrier-Grade NAT (CGNAT)** dan tidak bisa menerima koneksi masuk secara langsung, Termux membuat reverse tunnel ke VPS. Client terhubung ke port VPS, lalu traffic diteruskan ke Termux melalui reverse tunnel, dan **traffic keluar (egress) ke Internet tetap 100% melalui SIM Card Android**.

> **PENTING TENTANG DATA VALIDITY, EGRESS IP & REAL-TIME TRAFFIC USAGE**:
> - Public IP VPS **bukanlah** IP proxy. Egress IP selalu diverifikasi langsung melalui koneksi proxy itu sendiri. Jika hasil IP tidak dapat diverifikasi atau terjadi IP conflict antar IP-check provider, status tidak akan dipalsukan menjadi `ONLINE`, melainkan ditandai `UNVERIFIED` atau `IP_CONFLICT`.
> - Penggunaan kuota / data proxy dihitung langsung pada socket level di SOCKS5 Agent (Termux) secara presisi (`bytes_in` / download & `bytes_out` / upload). Statistik ditransmisikan via SSE (Server-Sent Events) ke Web Dashboard secara real-time.

---

## 2. Struktur Direktori Project

```
proxy-manager/
├── backend/                  # REST API, SSE, Scheduler, Core Services
│   ├── src/
│   │   ├── routes/           # Auth, Proxies, Agents, Logs, Status, Analytics
│   │   ├── services/         # HealthChecker, StatusEngine, IPDetection, Rotation
│   │   ├── middleware/       # JWT Auth, Rate Limiter, Express Validator
│   │   ├── utils/            # DB connector, Logger, Helpers, Migrations
│   │   └── index.js          # Entrypoint server Express
│   └── package.json
├── frontend/                 # Web Dashboard
│   └── dist/                 # Static SPA (HTML5, Modern CSS, JS with SSE)
├── agent/                    # Termux Agent (Python)
│   ├── agent.py              # Main Termux Agent process
│   ├── network_controller.py # Multi-adapter network reconnect (Root/nmcli/Termux)
│   ├── ip_detector.py        # Multi-provider egress IP detector
│   ├── proxy_server.py       # SOCKS5 lifecycle manager
│   └── socks5_server.py      # Embedded Python SOCKS5 with authentication & byte counter
├── tunnel/                   # Reverse SSH/Autossh Tunnel for CGNAT bypass
│   ├── start_tunnel.sh
│   └── stop_tunnel.sh
├── database/
│   └── migrations/           # SQL migration scripts (001, 002)
├── cli/
│   └── proxy-manager         # CLI Management tool
├── nginx/
│   └── proxy-manager.conf    # Nginx reverse proxy configuration
├── systemd/
│   └── proxy-manager-backend.service
├── install.sh                # Universal VPS Auto-Installer
├── install-agent.sh          # Universal Termux Auto-Installer
├── uninstall.sh              # Clean Uninstallation script
├── backup.sh                 # Database & config backup tool
├── restore.sh                # Restore tool
└── README.md
```

---

## 3. TUTORIAL LENGKAP: SETUP DARI NOL SAMPAI PROXY AKTIF & BISA DIGUNAKAN

Ikuti langkah-langkah berikut secara berurutan:

### LANGKAH 1: Setup VPS (Server Control Plane)

1. Masuk ke VPS Anda via SSH sebagai `root`:
   ```bash
   ssh root@IP_VPS_ANDA
   ```
2. Clone repository atau masuk ke folder `proxy-manager`:
   ```bash
   cd /home/ubuntu/proxy-manager
   ```
3. Jalankan script auto installer VPS:
   ```bash
   sudo ./install.sh
   ```
   *(Atau jalankan non-interaktif: `sudo ./install.sh --user admin --password "PasswordKuat123" -y`)*
4. Setelah selesai, installer akan menampilkan:
   - Dashboard URL: `http://IP_VPS_ANDA/`
   - Admin Username & Password
5. Buka browser di komputer/laptop Anda dan akses `http://IP_VPS_ANDA/`. Login dengan akun admin.

---

### LANGKAH 2: Hubungkan Android / Termux (Pilihan: Data Seluler atau WiFi)

1. Buka **Web Dashboard** di browser, klik menu **📱 Termux Agents**, lalu klik tombol **+ Pair New Agent**.
2. Dashboard akan menampilkan perintah *One-Liner Auto-Install* lengkap dengan token pairing (berlaku 15 menit). Salin perintah tersebut.
3. Buka aplikasi **Termux** di HP Android Anda.
4. Anda dapat memilih menggunakan **Data Internet Seluler** ATAU **WiFi**:

#### Opsi A: Menggunakan Data Internet Seluler (SIM Card / 4G / 5G) [Rekomendasi]
Pastikan paket data seluler aktif, lalu jalankan di Termux:
```bash
curl -fsSL http://IP_VPS_ANDA/install-agent.sh | bash -s -- http://IP_VPS_ANDA TOKEN_PAIRING "HP-Seluler-01" cellular
```
*Karakteristik: IP berganti setiap rotasi mode pesawat, terdeteksi sebagai Mobile Carrier (Telkomsel, Indosat, XL, Tri).*

#### Opsi B: Menggunakan Koneksi WiFi (WiFi Rumah / Kantor / Hotspot)
Pastikan HP terhubung ke jaringan WiFi Anda, lalu jalankan di Termux:
```bash
curl -fsSL http://IP_VPS_ANDA/install-agent.sh | bash -s -- http://IP_VPS_ANDA TOKEN_PAIRING "HP-WiFi-01" wifi
```
*Karakteristik: Kecepatan tinggi dan kuota tidak terbatas (unlimited). Public IP proxy akan menggunakan IP publik dari ISP WiFi Anda (IndiHome, Biznet, MyRepublic, dll.).*

#### Opsi C: Mode Interaktif (Pilih Menu Manual)
Jika dijalankan tanpa parameter jaringan:
```bash
curl -fsSL http://IP_VPS_ANDA/install-agent.sh | bash
```
*Installer akan menampilkan menu interaktif di layar Termux untuk memilih antara Data Seluler atau WiFi.*

5. Script akan secara otomatis:
   - Menginstall paket Python, OpenSSH, Autossh, Curl, Tar.
   - Mengaktifkan **Wake-Lock** agar Termux tidak dimatikan Android saat layar mati.
   - Melakukan pairing dengan VPS secara aman sesuai mode jaringan yang dipilih (Seluler / WiFi).
   - **Langsung Aktif di Background**: SOCKS5 Server dan Agent langsung otomatis berjalan tanpa perlu Anda jalankan manual.
   - **Auto-Restart Watchdog Supervisor (`proxy-supervisor.sh`)**: Pengawas background otomatis aktif untuk memantau proses setiap 4 detik. Jika aplikasi tertutup atau crash karena RAM cleaner Android, supervisor akan **langsung menyalakan kembali proses secara otomatis**.
   - Menginstall hook **Termux:Boot** (agar otomatis berjalan kembali saat HP direstart / mati lalu dinyalakan kembali).

6. Perintah Manajemen di Termux:
   - Cek Status Service & Watchdog: `~/status-agent.sh`
   - Restart Agent: `~/restart-agent.sh`
   - Stop Agent & Watchdog: `~/stop-agent.sh`
   - Start Tunnel Relay: `~/start-tunnel.sh IP_VPS 22 root 10001 10001`
   - Stop Tunnel Relay: `~/stop-tunnel.sh`
   - Lihat Log Real-time: `tail -f ~/proxy-agent/agent.log`

7. Kembali ke Web Dashboard di browser, refresh halaman: Anda akan melihat Agent Android Anda telah berstatus **ONLINE** dengan Public IP terverifikasi!

---

### LANGKAH 3: Buat Proxy di Web Dashboard

1. Di Web Dashboard, klik menu **🌐 Proxies**, lalu klik tombol **+ Add New Proxy**.
2. Isi form:
   - **Proxy Name**: Misal `Mobile-Proxy-01`
   - **Listen Port**: Misal `10001` *(pastikan port range 10000-10100 dibuka di VPS)*
   - **Bind to Termux Agent**: Pilih perangkat Android yang baru saja di-pair (`HP-Saya-01`).
   - **Proxy Username**: Misal `userproxy1`
   - **Proxy Password**: Password untuk proxy client (minimal 8 karakter, misal `PassRahasia123`).
3. Klik **Create Proxy**. Proxy baru Anda sekarang terdaftar di sistem.

---

### LANGKAH 4: Hubungkan Tunnel Traffic (Mode B / Reverse Relay)

Karena hampir semua operator seluler menggunakan CGNAT (tidak mengizinkan koneksi langsung dari internet ke HP), buka reverse tunnel agar port `10001` di VPS otomatis me-relay traffic ke SOCKS5 di Termux Android:

Di Termux Android, jalankan cukup satu baris:
```bash
~/start-tunnel.sh IP_VPS_ANDA 22 root 10001 10001
```
*(Script ini otomatis menghasilkan SSH key jika belum ada, menyambungkan reverse tunnel dengan keep-alive auto-reconnect, dan installer VPS sudah otomatis mengaktifkan `GatewayPorts yes` di SSH daemon agar port 10001 VPS dapat diakses publik dari luar).*

Untuk mematikan tunnel:
```bash
~/stop-tunnel.sh
```

---

### LANGKAH 5: Cara Menggunakan Proxy di Aplikasi & Browser

Proxy Anda sekarang telah aktif dan siap digunakan di mana saja dengan format:
- **Host**: `IP_VPS_ANDA`
- **Port**: `10001`
- **Type**: `SOCKS5`
- **Username**: `userproxy1`
- **Password**: `PassRahasia123`

#### A. Uji Menggunakan cURL (Terminal / CMD)
Buka terminal di komputer Anda dan jalankan:
```bash
curl -x socks5h://userproxy1:PassRahasia123@IP_VPS_ANDA:10001 https://api.ipify.org
```
*Catatan: Gunakan `socks5h://` agar resolusi DNS dilakukan melalui HP seluler.*
**Hasil**: Terminal akan mencetak alamat IP seluler HP Anda (bukan IP VPS)!

#### B. Menggunakan di Browser (Chrome / Firefox / Brave)
Gunakan ekstensi seperti **FoxyProxy Standard** atau **Proxy SwitchyOmega**:
1. Pasang ekstensi FoxyProxy di browser.
2. Tambahkan proxy baru:
   - Proxy Type: **SOCKS5**
   - Server IP: `IP_VPS_ANDA`
   - Port: `10001`
   - Username: `userproxy1`
   - Password: `PassRahasia123`
   - Centang opsi: **Send DNS through SOCKS5 proxy**.
3. Aktifkan proxy tersebut di browser, lalu buka situs [https://whoer.net](https://whoer.net) atau [https://ipinfo.io](https://ipinfo.io).
4. Anda akan melihat ISP Anda terdeteksi sebagai operator seluler (Telkomsel, XL, Indosat, dll.)!

#### C. Menggunakan di Python (Scripting / Bot)
```python
import requests

proxies = {
    'http': 'socks5h://userproxy1:PassRahasia123@IP_VPS_ANDA:10001',
    'https': 'socks5h://userproxy1:PassRahasia123@IP_VPS_ANDA:10001',
}

resp = requests.get('https://api.ipify.org?format=json', proxies=proxies, timeout=10)
print("IP Mobile Proxy Saya:", resp.json()['ip'])
```

---

### LANGKAH 6: Cara Melakukan Rotasi IP (Mengganti IP Seluler)

1. **Rotasi Manual via Web Dashboard**:
   - Buka Web Dashboard -> menu **Proxies**.
   - Klik tombol **🔄 Rotate** pada proxy yang ingin dirotasi.
   - Agent di Android akan mematikan dan menyalakan kembali data seluler (Airplane mode / reset radio).
   - Tunggu 5-10 detik: Web Dashboard akan otomatis mendeteksi IP baru dan mencatat perubahannya di menu **IP History**!
2. **Rotasi Otomatis Berkala (Scheduled Auto-Rotation)**:
   - Masuk ke menu **Settings** di Web Dashboard.
   - Atur `Default Rotation Interval (seconds)` misalnya `600` (setiap 10 menit).
   - Sistem akan secara otomatis mengganti IP seluler Anda secara berkala tanpa intervensi manual.

---

## 4. Pengaturan Latar Belakang Android (PENTING)

Agar Termux tidak dimatikan oleh sistem operasi Android saat layar mati:
1. **Matikan Battery Optimization**:
   Buka *Pengaturan HP -> Aplikasi -> Termux -> Baterai -> Pilih "Tidak Dibatasi" (Unrestricted / Don't optimize)*.
2. **Lock Aplikasi di Recent Apps**:
   Buka menu Recent Apps di HP Anda, tahan ikon Termux, dan klik ikon **Gembok (Lock)** agar tidak terhapus saat membersihkan RAM.
3. **Aktifkan Wake Lock**:
   Installer Termux sudah otomatis memanggil `termux-wake-lock`.

---

## 5. Fitur Stabilitas & Self-Healing (Enterprise Grade)

Sistem ini telah dilengkapi mekanisme pertahanan berlapis untuk menjamin kestabilan jangka panjang:

1. **Watchdog Pemulihan Otomatis (Self-Healing)**:
   - Termux Agent memiliki watchdog internal yang terus memantau status server SOCKS5. Jika server SOCKS5 mati karena memory pressure Android, agent otomatis menyalakan ulang SOCKS5 tanpa mengganggu status online ke VPS.
2. **Proteksi Concurrency & Anti-Memory Leak**:
   - Server SOCKS5 menggunakan `threading.Semaphore` dengan batas maksimum 256 koneksi konkuren bersamaan untuk mencegah kehabisan file descriptor (FD limit) pada kernel Android.
   - Menggunakan opsi `TCP_NODELAY` dan penutupan soket otomatis dalam blok `finally`.
3. **Resilient Network Flap & Rotation Recovery**:
   - Saat rotasi IP mematikan paket data seluler sementara, agent secara cerdas menahan pelaporan ke VPS, menunggu hingga koneksi internet seluler pulih sepenuhnya, lalu mengirimkan hasil rotasi dengan retry berulang.
4. **Auto OpenSSH GatewayPorts Config**:
   - Script installer VPS secara otomatis menyetel `GatewayPorts yes` pada SSH daemon VPS, memastikan relay port binding reverse tunnel dapat diakses dari internet publik.
5. **Database Concurrency Protection**:
   - SQLite dikonfigurasi dengan WAL mode (`journal_mode=WAL`) dan `busy_timeout=5000ms`, menjamin kueri data besar dari dashboard tidak mengunci penulisan data real-time heartbeat.
6. **Background Resilience**:
   - Systemd service VPS dikonfigurasi dengan restart otomatis (`RestartSec=3`) dan peningkatan batas `LimitNOFILE=65535`.

---

## 6. Manajemen & CLI VPS

Anda dapat mengelola seluruh sistem melalui terminal VPS menggunakan CLI `proxy-manager`:

```bash
# Status & Diagnostik
proxy-manager status
proxy-manager test

# Melihat data proxy & agent
proxy-manager proxy list
proxy-manager agent list
proxy-manager ip
proxy-manager logs

# Kontrol Service
proxy-manager restart
proxy-manager stop
proxy-manager start

# Backup & Restore
proxy-manager backup
proxy-manager restore backups/backup_YYYYMMDD_HHMMSS.tar.gz

# Uninstall Bersih
proxy-manager uninstall
```

---

## 7. Pengujian Sistem (Automated Tests)

Untuk menjalankan seluruh test suite otomatis:

```bash
cd /home/ubuntu/proxy-manager/backend
npm test
```
*Hasil pengujian mencakup verifikasi Auth, Status Engine, IP Detection, Traffic Usage, SOCKS5 Handshake, dan Agent Pairing.*
