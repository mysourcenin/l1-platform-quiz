# 🖥️ RHCSA Lab Environment — Tutorial Lengkap dari Nol
## Laptop Windows (Intel/AMD x86_64) + VMware Workstation + Rocky Linux 9.8 x86_64

> Tutorial ini dirancang khusus untuk **pemula absolut** yang baru belajar Linux.
> Setiap command dijelaskan: **apa fungsinya**, **menggunakan user apa**, dan **kenapa harus dijalankan**.
>
> **Platform**: Laptop Windows · VMware Workstation · Rocky Linux 9.8 **x86_64**

---

## Daftar Isi

1. [Arsitektur Lab & Kebutuhan VM](#1-arsitektur-lab--kebutuhan-vm)
2. [Tahap 1 — Setup VMware & Instalasi Rocky Linux](#2-tahap-1--setup-utm--instalasi-rocky-linux)
3. [Tahap 2 — Linux Fundamental & Service Triage](#3-tahap-2--linux-fundamental--service-triage)
4. [Tahap 3 — Runtime, Web Server, Monitoring & Storage](#4-tahap-3--runtime-web-server-monitoring--storage)
5. [Tahap 4 — Containerization & SOP Escalation](#5-tahap-4--containerization--sop-escalation)
6. [Latihan Mandiri (Self-Assessment)](#6-latihan-mandiri)

---

## 1. Arsitektur Lab & Kebutuhan VM

### Mengapa Butuh Lebih dari 1 VM?

Dalam dunia kerja nyata (enterprise), kamu akan mengelola **banyak server sekaligus** — server web, database, monitoring, dll. Lab ini mensimulasikan skenario tersebut.

### Jumlah VM yang Dibutuhkan: **3 VM**

| VM | Hostname | Peran | vCPU | RAM | Disk | IP (Contoh) |
|---|---|---|---|---|---|---|
| **VM-1** | `server1.lab.local` | Server utama (Web Server, Service Triage, SELinux, LVM, Cron, NFS Server) | 2 | 2 GB | **20 GB + 5 GB** (disk tambahan untuk LVM) | `192.168.64.10` |
| **VM-2** | `server2.lab.local` | Server kedua (NFS Client, AutoFS, Tuned, User/Group Management, Container Host) | 2 | 2 GB | **20 GB** | `192.168.64.11` |
| **VM-3** | `server3.lab.local` | Server database & monitoring target (MySQL/MariaDB, Prometheus Node Exporter target, Firewall rules) | 1 | 1.5 GB | **15 GB** | `192.168.64.12` |

> [!IMPORTANT]
> **Minimum RAM Laptop**: 16 GB (agar bisa menjalankan 3 VM + macOS bersamaan).
> Jika RAM hanya 8 GB, cukup buat **2 VM** dulu (VM-1 dan VM-2), VM-3 bisa dibuat nanti secara bergantian.

### Spesifikasi Detail per VM

#### VM-1: `server1.lab.local` — Server Utama
```
Tujuan     : Latihan RHCSA inti — web server, partisi, LVM, NFS, cron, SELinux
vCPU       : 2 core
RAM        : 2048 MB (2 GB)
Disk 1     : 20 GB (untuk OS Rocky Linux 9.8)
Disk 2     : 5 GB (untuk latihan partisi, LVM, swap)
Network    : Shared Network (VMware NAT, mendapat IP otomatis)
ISO        : Rocky-9.8-x86_64-dvd.iso
Hypervisor : VMware Workstation / VirtualBox
```

#### VM-2: `server2.lab.local` — Server Pendukung
```
Tujuan     : Client NFS, AutoFS, container host, user management
vCPU       : 2 core
RAM        : 2048 MB (2 GB)
Disk 1     : 20 GB (untuk OS)
Network    : Shared Network (sama subnet dengan VM-1)
ISO        : Rocky-9.8-x86_64-dvd.iso
Hypervisor : VMware Workstation / VirtualBox
```

#### VM-3: `server3.lab.local` — Database & Target Monitoring
```
Tujuan     : Database server, latihan firewall, target monitoring
vCPU       : 1 core
RAM        : 1536 MB (1.5 GB)
Disk 1     : 15 GB (untuk OS)
Network    : Shared Network (sama subnet)
ISO        : Rocky-9.8-x86_64-dvd.iso
Hypervisor : VMware Workstation / VirtualBox
```

### Download ISO Rocky Linux 9.8 (x86_64)

> [!IMPORTANT]
> Laptop Windows pada umumnya menggunakan prosesor **Intel atau AMD**. Kamu **WAJIB** download versi **x86_64**.

1. Buka browser → [https://rockylinux.org/download](https://rockylinux.org/download)
2. Pilih **Rocky Linux 9.8**
3. Pilih arsitektur: **x86_64** ← yang ini!
4. Download file **DVD ISO** (~9-10 GB) — berisi semua package offline
5. Simpan file `Rocky-9.8-x86_64-dvd.iso` di folder yang mudah ditemukan (misal `~/Downloads`)

---

## 2. Tahap 1 — Setup VMware & Instalasi Rocky Linux

### 2.1 Install VMware Workstation di Windows

```
📍 Posisi: Di Windows (bukan di dalam VM)
👤 User: User biasa Windows kamu
```

1. Download **VMware Workstation Player** (gratis untuk penggunaan personal) dari situs resmi Broadcom/VMware.
2. Atau gunakan **VirtualBox** jika lebih familiar: [https://www.virtualbox.org/](https://www.virtualbox.org/)
3. Install aplikasi seperti biasa di Windows (Next -> Next -> Finish).

### 2.2 Membuat VM-1 (`server1.lab.local`)

```
📍 Posisi: Di aplikasi VMware (Windows)
👤 User: User biasa Windows kamu
```

**Step-by-step di VMware Workstation:**

1. Buka UTM → klik tombol **"+"** (Create a New Virtual Machine)
2. Pilih **"Virtualize"** ← wajib pilih ini untuk M1 (bukan "Emulate"!)
3. Pilih **"Linux"**
4. **Boot ISO Image** → klik **Browse** → pilih file `Rocky-9.8-x86_64-dvd.iso`
5. Konfigurasi Hardware:
   - **Memory**: `2048` MB
   - **CPU Cores**: `2`
6. Storage:
   - **Size**: `20` GB
7. Name: `server1-lab`
8. Klik **Finish**

**Menambahkan Disk Kedua (5 GB) untuk VM-1:**

1. Klik VM `server1-lab` → **Edit virtual machine settings**
2. Klik **Add...** → pilih **Hard Disk**
3. Disk Type: **SCSI** atau **NVMe** (direkomendasikan)
4. Create a new virtual disk → Size: **5 GB**
5. Klik **Finish** lalu **OK**

**Konfigurasi Network:**

1. Di Settings VM → klik **Network Adapter**
2. Network connection: pilih **NAT: Used to share the host's IP address**
3. Klik **OK**

### 2.3 Instalasi Rocky Linux 9.8 (VM-1)

```
📍 Posisi: Di dalam jendela VMware (booting dari ISO)
👤 User: Installer Anaconda (GUI installer Red Hat)
```

1. **Start VM** → klik tombol ▶️ Play
2. Pada boot menu, pilih: **Install Rocky Linux 9.8**
3. Pilih bahasa: **English (United States)** → Continue

**Installation Summary — Yang WAJIB dikonfigurasi:**

#### a. Keyboard Layout
- Biarkan default **English (US)** → Done

#### b. Time & Date
- Pilih **Asia/Jakarta** (atau timezone kamu) → Done

#### c. Installation Source
- Biarkan **Local media** (karena pakai DVD ISO) → Done

#### d. Software Selection
- Pilih **"Server with GUI"** (supaya ada desktop untuk belajar, opsional)
- Atau pilih **"Server"** (tanpa GUI, lebih ringan — **RECOMMENDED** untuk lab)
- Di kolom kanan, centang:
  - ✅ Development Tools
  - ✅ System Tools
  - ✅ Container Management (untuk latihan Docker/Podman)
- Klik **Done**

#### e. Installation Destination
- Pilih disk **20 GB** (disk utama)
- Storage Configuration: pilih **"Automatic"**
- ⚠️ **JANGAN** pilih disk 5 GB — biarkan disk itu kosong untuk latihan partisi nanti
- Klik **Done**

#### f. Network & Hostname
- Klik toggle **ON** pada interface network (misal `enp0s1`)
- Hostname: ketik `server1.lab.local`
- Klik **Apply** → **Done**

#### g. Root Password
- Set password root: `RedHat123!` (atau password kuat pilihan kamu)
- ✅ Centang **"Allow root login with password"**
- Klik **Done**

#### h. User Creation
- Full Name: `student`
- Username: `student`
- Password: `student123`
- ✅ Centang **"Make this user administrator"** (menambahkan ke group `wheel` untuk akses `sudo`)
- Klik **Done**

4. Klik **Begin Installation** → tunggu selesai (~5-15 menit)
5. Setelah selesai → klik **Reboot System**

> [!IMPORTANT]
> Setelah reboot, **lepaskan ISO** dari VM agar tidak boot ulang ke installer:
> Di VMware → Settings VM → CD/DVD → hapus centang pada **Connect at power on**

### 2.4 Login Pertama Kali ke VM-1

```
📍 Posisi: Di dalam VM (console VMware)
👤 User: root (pertama kali) lalu student
```

Setelah reboot, kamu akan melihat login prompt:

```
Rocky Linux 9.8 (Blue Onyx)
Kernel 5.14.0-503.x.el9.x86_64 on an x86_64

server1 login: _
```

**Login sebagai root:**
```bash
# Ketik username
server1 login: root
# Ketik password (tidak terlihat saat diketik — ini normal!)
Password: RedHat123!
```

**Penjelasan:**
- `root` = **superuser** / administrator tertinggi di Linux. Punya akses PENUH ke seluruh sistem
- Password tidak ditampilkan saat diketik = fitur keamanan Linux

### 2.5 Konfigurasi Awal Wajib (Post-Install) — VM-1

```
📍 Posisi: Di dalam VM-1
👤 User: root
```

#### a. Verifikasi hostname

```bash
hostnamectl
```

**Penjelasan**: `hostnamectl` menampilkan informasi hostname (nama komputer) dan OS. Pastikan menampilkan `server1.lab.local`.

Jika hostname belum benar:
```bash
hostnamectl set-hostname server1.lab.local
```

**Penjelasan**: `hostnamectl set-hostname` mengubah nama hostname secara permanen.

#### b. Cek IP Address

```bash
ip addr show
```

**Penjelasan**: `ip addr show` (atau singkatnya `ip a`) menampilkan semua interface jaringan dan IP address yang terpasang. Catat IP address-nya (misal: `192.168.64.10`).

#### c. Update sistem (opsional tapi direkomendasikan)

```bash
dnf update -y
```

**Penjelasan**:
- `dnf` = **D**andified **Y**um — package manager untuk Rocky/RHEL 9. Digunakan untuk install, update, hapus software
- `update` = sub-command untuk mengupdate semua package ke versi terbaru
- `-y` = otomatis jawab "yes" untuk semua konfirmasi (supaya tidak perlu ketik `y` berkali-kali)

#### d. Tambahkan entri /etc/hosts (supaya VM bisa saling kenal)

```bash
cat >> /etc/hosts << 'EOF'
192.168.64.10  server1.lab.local  server1
192.168.64.11  server2.lab.local  server2
192.168.64.12  server3.lab.local  server3
EOF
```

**Penjelasan**:
- `cat >> /etc/hosts` = menambahkan teks ke AKHIR file `/etc/hosts` (tanda `>>` berarti append/tambah, bukan overwrite)
- `<< 'EOF'` ... `EOF` = "here document" — cara menulis teks multi-baris langsung di terminal
- `/etc/hosts` = file yang memetakan IP ke hostname. Ini seperti "buku telepon" lokal — Linux akan cek file ini dulu sebelum bertanya ke DNS server

> [!TIP]
> Sesuaikan IP address di atas dengan IP yang kamu dapatkan dari `ip addr show`.

#### e. Cek koneksi internet

```bash
ping -c 3 google.com
```

**Penjelasan**:
- `ping` = mengirim paket ICMP ke tujuan untuk cek koneksi jaringan
- `-c 3` = kirim 3 paket saja (tanpa `-c`, ping akan jalan terus sampai di-stop Ctrl+C)
- Jika sukses, artinya VM bisa akses internet

#### f. Aktifkan Cockpit (Web Console — Opsional tapi berguna)

```bash
systemctl enable --now cockpit.socket
```

**Penjelasan**:
- `systemctl` = perintah untuk mengelola service/daemon di Linux (systemd)
- `enable` = membuat service start otomatis setiap kali reboot
- `--now` = sekaligus start service SEKARANG juga (jadi tidak perlu 2 command terpisah)
- `cockpit.socket` = web-based management console. Bisa diakses via browser di `https://IP-VM:9090`

### 2.6 Membuat VM-2 dan VM-3

Ulangi langkah **2.2** dan **2.3** untuk membuat VM-2 dan VM-3 dengan perbedaan:

| Parameter | VM-2 | VM-3 |
|---|---|---|
| Name di VMware | `server2-lab` | `server3-lab` |
| RAM | 2048 MB | 1536 MB |
| CPU | 2 | 1 |
| Disk | 20 GB (1 disk saja) | 15 GB (1 disk saja) |
| Hostname | `server2.lab.local` | `server3.lab.local` |
| Root password | `RedHat123!` | `RedHat123!` |
| User student | sama | sama |

> [!IMPORTANT]
> Jangan lupa tambahkan entri `/etc/hosts` yang SAMA di setiap VM agar mereka bisa saling ping berdasarkan hostname.

**Verifikasi antar-VM bisa komunikasi (dari VM-1):**

```bash
# 📍 Posisi: VM-1 | 👤 User: root
ping -c 2 server2.lab.local
ping -c 2 server3.lab.local
```

---

## 3. Tahap 2 — Linux Fundamental & Service Triage

> Tahap ini sesuai dengan **Modul 1: Linux Fundamental & Triage** dan **Modul 2: Runtime & Web Server** dari lab assessment.

### 3.1 Manajemen User & Group

```
📍 Posisi: VM-1 (server1.lab.local)
👤 User: root
```

#### Membuat User Baru

```bash
useradd -m -s /bin/bash sysadmin
```

**Penjelasan**:
- `useradd` = perintah untuk membuat user baru
- `-m` = buat home directory untuk user (`/home/sysadmin`)
- `-s /bin/bash` = set default shell ke Bash (supaya user bisa mengetik command)
- `sysadmin` = nama user yang dibuat

```bash
passwd sysadmin
```

**Penjelasan**: `passwd` = mengubah/membuat password untuk user. Kamu akan diminta ketik password 2 kali.

#### Membuat Group dan Menambahkan User

```bash
groupadd devops
```

**Penjelasan**: `groupadd` = membuat group baru bernama `devops`. Group digunakan untuk mengatur permission secara kolektif.

```bash
usermod -aG devops sysadmin
usermod -aG devops student
```

**Penjelasan**:
- `usermod` = memodifikasi user yang sudah ada
- `-aG devops` = **a**ppend (tambahkan) user ke **G**roup `devops`. Flag `-a` penting! Tanpa `-a`, user akan DIKELUARKAN dari group lain
- Artinya: user `sysadmin` dan `student` sekarang anggota group `devops`

#### Verifikasi

```bash
id sysadmin
```

**Penjelasan**: `id` = menampilkan UID (User ID), GID (Group ID), dan daftar group dari user. Output contoh:
```
uid=1001(sysadmin) gid=1001(sysadmin) groups=1001(sysadmin),1002(devops)
```

```bash
cat /etc/passwd | grep sysadmin
```

**Penjelasan**:
- `cat /etc/passwd` = menampilkan isi file daftar semua user di sistem
- `| grep sysadmin` = pipe (|) menyalurkan output ke `grep` yang memfilter hanya baris mengandung "sysadmin"

### 3.2 Manajemen File Permission & Ownership

```
📍 Posisi: VM-1
👤 User: root
```

#### Membuat Struktur Direktori

```bash
mkdir -p /opt/webapp/public
mkdir -p /opt/webapp/config
mkdir -p /opt/webapp/logs
```

**Penjelasan**:
- `mkdir` = **m**ake **dir**ectory — membuat folder baru
- `-p` = membuat parent directory juga jika belum ada (contoh: jika `/opt/webapp` belum ada, otomatis dibuat)

#### Mengubah Ownership

```bash
chown -R sysadmin:devops /opt/webapp
```

**Penjelasan**:
- `chown` = **ch**ange **own**ership — mengubah pemilik file/folder
- `-R` = recursive — terapkan ke semua isi folder di dalamnya
- `sysadmin:devops` = user `sysadmin` sebagai owner, group `devops` sebagai group owner
- `/opt/webapp` = target folder

#### Mengatur Permission

```bash
chmod 750 /opt/webapp
chmod 755 /opt/webapp/public
chmod 640 /opt/webapp/config
chmod 660 /opt/webapp/logs
```

**Penjelasan**:
- `chmod` = **ch**ange **mod**e — mengubah permission (hak akses) file/folder
- Angka 3 digit = permission untuk [Owner][Group][Others]
- Setiap digit adalah jumlah: **r=4** (read), **w=2** (write), **x=1** (execute)

| Angka | Permission | Artinya |
|---|---|---|
| `7` | rwx | Baca + Tulis + Eksekusi |
| `5` | r-x | Baca + Eksekusi |
| `4` | r-- | Baca saja |
| `6` | rw- | Baca + Tulis |
| `0` | --- | Tidak ada akses |

Jadi `chmod 750 /opt/webapp` artinya:
- Owner (sysadmin): rwx (full access)
- Group (devops): r-x (bisa baca & masuk, tidak bisa tulis)
- Others: --- (tidak ada akses sama sekali)

#### Verifikasi Permission

```bash
ls -la /opt/webapp/
```

**Penjelasan**:
- `ls` = **l**i**s**t — menampilkan daftar file/folder
- `-l` = format panjang (menampilkan permission, owner, size, tanggal)
- `-a` = tampilkan file hidden juga (file dengan awalan titik)

Output contoh:
```
drwxr-x--- 5 sysadmin devops 4096 Aug 21 10:00 .
drwxr-xr-x 3 root     root   4096 Aug 21 10:00 ..
drwxr-xr-x 2 sysadmin devops 4096 Aug 21 10:00 public
drw-r----- 2 sysadmin devops 4096 Aug 21 10:00 config
drw-rw---- 2 sysadmin devops 4096 Aug 21 10:00 logs
```

### 3.3 Install & Kelola Web Server (Nginx)

```
📍 Posisi: VM-1
👤 User: root
```

#### Install Nginx

```bash
dnf install -y nginx
```

**Penjelasan**: Install package `nginx` (web server) dari repository. Flag `-y` = auto-confirm.

#### Aktifkan dan Start Nginx

```bash
systemctl enable --now nginx
```

**Penjelasan**: Aktifkan Nginx agar auto-start saat boot, sekaligus start sekarang.

#### Cek Status Service

```bash
systemctl status nginx
```

**Penjelasan**: Menampilkan status terkini dari service Nginx. Perhatikan:
- `Active: active (running)` = ✅ service berjalan normal
- `Active: inactive (dead)` = ❌ service mati
- `Active: failed` = ❌ service gagal start (ada error)

Output contoh (normal):
```
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled)
     Active: active (running) since Thu 2026-08-21 10:15:30 WIB
```

#### Buka Firewall untuk HTTP

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

**Penjelasan**:
- `firewall-cmd` = perintah untuk mengelola firewalld (firewall bawaan RHEL/Rocky)
- `--permanent` = perubahan disimpan secara permanen (tetap aktif setelah reboot)
- `--add-service=http` = mengizinkan traffic HTTP (port 80)
- `--add-service=https` = mengizinkan traffic HTTPS (port 443)
- `--reload` = muat ulang konfigurasi firewall agar perubahan berlaku

#### Verifikasi Web Server Berjalan

```bash
curl -I http://localhost
```

**Penjelasan**:
- `curl` = command-line tool untuk transfer data via URL (HTTP client)
- `-I` = hanya tampilkan HTTP response header (tanpa body/konten)
- `http://localhost` = mengakses web server di mesin sendiri

Output sukses:
```
HTTP/1.1 200 OK
Server: nginx/1.20.1
```

### 3.4 Live Log Tailing & Troubleshooting

```
📍 Posisi: VM-1
👤 User: root
```

#### Melihat Log Systemd (Journalctl)

```bash
journalctl -u nginx --no-pager -n 20
```

**Penjelasan**:
- `journalctl` = perintah untuk membaca log dari systemd journal
- `-u nginx` = filter log hanya untuk unit/service `nginx`
- `--no-pager` = tampilkan langsung tanpa pager (tidak perlu scroll)
- `-n 20` = tampilkan 20 baris terakhir saja

#### Live Tailing (Monitor Realtime)

```bash
journalctl -u nginx -f
```

**Penjelasan**:
- `-f` = **follow** — terus menampilkan log baru secara realtime (seperti `tail -f`)
- Tekan **Ctrl+C** untuk berhenti
- Ini sangat berguna saat troubleshooting — kamu bisa melihat error terjadi secara live

#### Melihat Log File Tradisional

```bash
tail -f /var/log/nginx/error.log
```

**Penjelasan**:
- `tail` = menampilkan bagian AKHIR file
- `-f` = follow — terus monitor file untuk baris baru
- `/var/log/nginx/error.log` = lokasi file log error Nginx

### 3.5 Pengecekan Network Socket & Port

```
📍 Posisi: VM-1
👤 User: root
```

#### Cek Port yang Sedang Listening

```bash
ss -tulpn
```

**Penjelasan setiap flag**:
- `ss` = **s**ocket **s**tatistics — menampilkan informasi koneksi jaringan (pengganti `netstat`)
- `-t` = tampilkan **TCP** sockets
- `-u` = tampilkan **UDP** sockets
- `-l` = hanya tampilkan socket yang **LISTENING** (menunggu koneksi)
- `-p` = tampilkan **process** yang menggunakan socket (nama aplikasi + PID)
- `-n` = tampilkan **numeric** (IP:port angka, bukan nama)

Output contoh:
```
State    Recv-Q Send-Q Local Address:Port  Peer Address:Port  Process
LISTEN   0      511    0.0.0.0:80          0.0.0.0:*          users:(("nginx",pid=1234,fd=6))
LISTEN   0      128    0.0.0.0:22          0.0.0.0:*          users:(("sshd",pid=890,fd=3))
```

#### Cek Port Spesifik

```bash
ss -tulpn | grep :80
```

**Penjelasan**: Filter output `ss` hanya menampilkan baris yang mengandung `:80` (port 80).

#### Cek Statistik Socket (untuk diagnosa socket exhaustion)

```bash
ss -s
```

**Penjelasan**: Menampilkan ringkasan statistik semua socket — berguna saat traffic tinggi untuk melihat jumlah koneksi TCP yang established.

### 3.6 Disk Space & Storage Management

```
📍 Posisi: VM-1
👤 User: root
```

#### Cek Penggunaan Disk per Mount Point

```bash
df -h
```

**Penjelasan**:
- `df` = **d**isk **f**ree — menampilkan ruang disk yang tersedia
- `-h` = **h**uman-readable — tampilkan ukuran dalam GB/MB (bukan bytes)

Output contoh:
```
Filesystem           Size  Used  Avail  Use%  Mounted on
/dev/mapper/rl-root  17G   3.5G  14G    21%   /
/dev/sda1            1.0G  210M  790M   21%   /boot
```

#### Cek Ukuran Folder Tertentu

```bash
du -sh /var/log
```

**Penjelasan**:
- `du` = **d**isk **u**sage — menghitung ukuran file/folder
- `-s` = **s**ummary — tampilkan total saja (bukan per-file)
- `-h` = human-readable

#### Lihat Disk yang Tersedia (termasuk disk belum dipartisi)

```bash
lsblk
```

**Penjelasan**: `lsblk` = **l**i**s**t **bl**oc**k** devices — menampilkan semua disk dan partisi dalam bentuk tree. Disk 5 GB tambahan akan muncul sebagai `sdb` (atau `sdb`) tanpa partisi.

Output contoh:
```
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             0:0    0   20G  0 disk 
├─sda1          0:1    0    1G  0 part /boot
└─sda2          0:2    0   19G  0 part 
  ├─rl-root   0:0    0   17G  0 lvm  /
  └─rl-swap   0:1    0    2G  0 lvm  [SWAP]
sdb             0:16   0    5G  0 disk          ← disk tambahan, belum dipartisi!
```

### 3.7 Partisi & LVM (Logical Volume Manager)

```
📍 Posisi: VM-1
👤 User: root
```

> [!CAUTION]
> Latihan partisi ini dilakukan pada **disk kedua (`/dev/sdb`)** yang kosong. **JANGAN** partisi disk utama (`/dev/sda`) karena bisa merusak OS!

#### Step 1: Buat Partisi pada Disk Kedua

```bash
fdisk /dev/sdb
```

**Penjelasan**: `fdisk` = tool partisi disk interaktif. Setelah dijalankan, kamu masuk ke "mode interaktif" fdisk.

Di dalam fdisk, ketik perintah berikut satu per satu:

```
Command: n          ← [n]ew partition
Partition type: p   ← [p]rimary partition
Partition number: 1 ← partisi ke-1
First sector: tekan Enter (default)
Last sector: +2G    ← ukuran 2 GB

Command: n          ← buat partisi kedua
Partition type: p
Partition number: 2
First sector: tekan Enter
Last sector: +2G    ← ukuran 2 GB lagi

Command: t          ← ubah [t]ype partisi
Partition number: 1
Hex code: 8e        ← 8e = Linux LVM

Command: t
Partition number: 2
Hex code: 8e

Command: w          ← [w]rite (simpan) dan keluar
```

#### Step 2: Buat Physical Volume (PV)

```bash
pvcreate /dev/sdb1 /dev/sdb2
```

**Penjelasan**: `pvcreate` = menandai partisi sebagai Physical Volume — "menyiapkan" partisi agar bisa digunakan oleh LVM.

#### Step 3: Buat Volume Group (VG)

```bash
vgcreate vg_data /dev/sdb1 /dev/sdb2
```

**Penjelasan**: `vgcreate` = membuat Volume Group bernama `vg_data` dari 2 PV. VG adalah "pool" penyimpanan — semua PV digabung menjadi 1 pool besar.

#### Step 4: Buat Logical Volume (LV)

```bash
lvcreate -L 3G -n lv_projects vg_data
```

**Penjelasan**:
- `lvcreate` = membuat Logical Volume (partisi virtual) dari VG
- `-L 3G` = ukuran 3 GB
- `-n lv_projects` = nama LV: `lv_projects`
- `vg_data` = ambil ruang dari VG ini

#### Step 5: Format dengan Filesystem

```bash
mkfs.xfs /dev/vg_data/lv_projects
```

**Penjelasan**: `mkfs.xfs` = **m**a**k**e **f**ile**s**ystem — format LV dengan filesystem XFS (default RHEL/Rocky).

#### Step 6: Mount (Pasang)

```bash
mkdir /mnt/projects
mount /dev/vg_data/lv_projects /mnt/projects
```

**Penjelasan**:
- `mkdir /mnt/projects` = buat folder sebagai titik mount
- `mount` = memasang filesystem ke direktori agar bisa diakses

#### Step 7: Buat Permanen (Auto-mount saat Boot)

```bash
echo '/dev/vg_data/lv_projects  /mnt/projects  xfs  defaults  0  0' >> /etc/fstab
```

**Penjelasan**:
- `/etc/fstab` = file konfigurasi mount permanen — Linux membaca file ini saat boot untuk auto-mount partisi
- Format kolom: `device  mountpoint  fstype  options  dump  fsck`

#### Verifikasi

```bash
df -h /mnt/projects
lvs
vgs
pvs
```

**Penjelasan**:
- `lvs` = menampilkan semua Logical Volume
- `vgs` = menampilkan semua Volume Group
- `pvs` = menampilkan semua Physical Volume

#### Bonus: Extend (Memperbesar) LV

```bash
lvextend -L +500M /dev/vg_data/lv_projects
xfs_growfs /mnt/projects
```

**Penjelasan**:
- `lvextend -L +500M` = tambah 500 MB ke LV
- `xfs_growfs` = perbesar filesystem XFS mengikuti ukuran LV baru (filesystem harus di-resize terpisah dari LV)

---

## 4. Tahap 3 — Runtime, Web Server, Monitoring & Storage

### 4.1 Manajemen Cron Job (Scheduled Tasks)

```
📍 Posisi: VM-1
👤 User: root
```

#### Membuat Cron Job

```bash
crontab -e
```

**Penjelasan**: `crontab -e` = **e**dit crontab (tabel penjadwalan) untuk user saat ini. Akan membuka editor teks.

Tambahkan baris berikut:

```
*/5 * * * * /usr/bin/systemctl status nginx >> /var/log/nginx-check.log 2>&1
0 2 * * * /usr/bin/find /tmp -type f -mtime +7 -delete
```

**Penjelasan format cron**:
```
┌───────────── menit (0-59)
│ ┌─────────── jam (0-23)
│ │ ┌───────── hari dalam bulan (1-31)
│ │ │ ┌─────── bulan (1-12)
│ │ │ │ ┌───── hari dalam minggu (0-7, 0 dan 7 = Minggu)
│ │ │ │ │
* * * * * command
```

- Baris 1: `*/5 * * * *` = setiap 5 menit, cek status nginx dan catat ke log
- Baris 2: `0 2 * * *` = setiap hari jam 2 pagi, hapus file di /tmp yang lebih tua dari 7 hari
- `2>&1` = redirect error output ke file yang sama

#### Verifikasi Cron Job

```bash
crontab -l
```

**Penjelasan**: `crontab -l` = **l**ist — menampilkan semua cron job untuk user saat ini.

### 4.2 SELinux (Security Enhanced Linux)

```
📍 Posisi: VM-1
👤 User: root
```

#### Cek Status SELinux

```bash
getenforce
```

**Penjelasan**: Menampilkan mode SELinux saat ini:
- `Enforcing` = aktif dan memblokir akses yang tidak sesuai policy
- `Permissive` = aktif tapi hanya mencatat pelanggaran (tidak memblokir)
- `Disabled` = nonaktif

```bash
sestatus
```

**Penjelasan**: Menampilkan informasi SELinux yang lebih detail (mode, policy, dll).

#### Mengubah Mode SELinux (Sementara)

```bash
setenforce 0
```

**Penjelasan**: `setenforce 0` = ubah ke mode Permissive (sementara, kembali ke Enforcing setelah reboot). Angka `1` = Enforcing.

#### Mengubah Mode SELinux (Permanen)

```bash
vi /etc/selinux/config
```

Ubah baris:
```
SELINUX=enforcing
```
menjadi:
```
SELINUX=permissive
```

#### Mengecek SELinux Context pada File

```bash
ls -Z /var/www/html/
```

**Penjelasan**: `ls -Z` = menampilkan SELinux security context pada file. Ini penting saat Nginx tidak bisa membaca file — mungkin SELinux context-nya salah.

#### Mengubah SELinux Context

```bash
chcon -t httpd_sys_content_t /var/www/html/index.html
```

**Penjelasan**:
- `chcon` = **ch**ange **con**text — mengubah SELinux context
- `-t httpd_sys_content_t` = set type ke `httpd_sys_content_t` (type yang diizinkan untuk web server)

#### Restore Default SELinux Context

```bash
restorecon -Rv /var/www/html/
```

**Penjelasan**: `restorecon` = mengembalikan SELinux context ke default sesuai policy. Flag `-R` = recursive, `-v` = verbose (tampilkan apa yang diubah).

### 4.3 NFS (Network File System) Server & Client

#### Di VM-1 (NFS Server)

```
📍 Posisi: VM-1
👤 User: root
```

```bash
dnf install -y nfs-utils
```

**Penjelasan**: Install paket NFS utilities.

```bash
mkdir -p /srv/nfs/shared
chmod 777 /srv/nfs/shared
echo "File dari NFS Server VM-1" > /srv/nfs/shared/readme.txt
```

**Penjelasan**: Buat folder yang akan di-share dan isi dengan file tes.

```bash
echo '/srv/nfs/shared  192.168.64.0/24(rw,sync,no_root_squash)' >> /etc/exports
```

**Penjelasan**:
- `/etc/exports` = file konfigurasi NFS — mendefinisikan folder mana yang di-share ke siapa
- `192.168.64.0/24` = izinkan akses dari seluruh subnet 192.168.64.x
- `rw` = read-write
- `sync` = tulis ke disk sebelum konfirmasi ke client
- `no_root_squash` = root di client tetap dianggap root di server (untuk lab saja!)

```bash
systemctl enable --now nfs-server
exportfs -rav
```

**Penjelasan**:
- Start service NFS server
- `exportfs -rav` = refresh daftar export NFS. Flag: `-r` re-export, `-a` all, `-v` verbose

```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --reload
```

**Penjelasan**: Buka firewall untuk layanan NFS.

#### Di VM-2 (NFS Client)

```
📍 Posisi: VM-2
👤 User: root
```

```bash
dnf install -y nfs-utils
mkdir -p /mnt/nfs-shared
mount -t nfs server1.lab.local:/srv/nfs/shared /mnt/nfs-shared
```

**Penjelasan**:
- `mount -t nfs` = mount tipe NFS
- `server1.lab.local:/srv/nfs/shared` = alamat NFS server dan path folder
- `/mnt/nfs-shared` = titik mount lokal

```bash
ls /mnt/nfs-shared/
cat /mnt/nfs-shared/readme.txt
```

**Penjelasan**: Verifikasi — kamu harusnya bisa melihat file dari VM-1!

Buat permanen:
```bash
echo 'server1.lab.local:/srv/nfs/shared  /mnt/nfs-shared  nfs  defaults  0  0' >> /etc/fstab
```

### 4.4 AutoFS (Auto-Mount on Demand)

```
📍 Posisi: VM-2
👤 User: root
```

```bash
dnf install -y autofs
```

```bash
echo '/mnt/auto  /etc/auto.nfs' >> /etc/auto.master
```

**Penjelasan**: `/etc/auto.master` = file utama AutoFS — menentukan "di mana mount" dan "file konfigurasi mana yang menjelaskan detailnya".

```bash
echo 'shared  -rw,sync  server1.lab.local:/srv/nfs/shared' > /etc/auto.nfs
```

**Penjelasan**: `/etc/auto.nfs` = file indirect map — saat folder `/mnt/auto/shared` diakses, otomatis mount NFS.

```bash
systemctl enable --now autofs
```

```bash
ls /mnt/auto/shared/
```

**Penjelasan**: Folder `/mnt/auto/shared` tidak terlihat di `ls /mnt/auto/` sampai kamu MENGAKSESNYA. AutoFS mount secara on-demand.

### 4.5 SSL Certificate Check

```
📍 Posisi: VM-1
👤 User: root
```

#### Membuat Self-Signed Certificate (untuk latihan)

```bash
mkdir -p /etc/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/server.key \
  -out /etc/nginx/ssl/server.crt \
  -subj "/CN=server1.lab.local"
```

**Penjelasan**:
- `openssl req` = generate certificate request
- `-x509` = buat self-signed cert (bukan request ke CA)
- `-nodes` = no DES — private key tanpa password
- `-days 365` = berlaku 365 hari
- `-newkey rsa:2048` = buat key RSA 2048-bit baru
- `-subj "/CN=server1.lab.local"` = Common Name = hostname server

#### Cek Informasi Certificate

```bash
openssl x509 -in /etc/nginx/ssl/server.crt -noout -dates
```

**Penjelasan**:
- `openssl x509` = tool untuk membaca X.509 certificate
- `-in` = file input certificate
- `-noout` = jangan tampilkan isi certificate (hanya info yang diminta)
- `-dates` = tampilkan tanggal `notBefore` (mulai berlaku) dan `notAfter` (expired)

Output:
```
notBefore=Aug 21 03:00:00 2026 GMT
notAfter=Aug 21 03:00:00 2027 GMT
```

### 4.6 Monitoring — Prometheus Node Exporter (Target di VM-3)

```
📍 Posisi: VM-3
👤 User: root
```

#### Install dan Jalankan Node Exporter

```bash
dnf install -y golang
```

Atau download binary langsung (ARM64 untuk Rocky aarch64):
```bash
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar xzf node_exporter-*.tar.gz
cp node_exporter-*/node_exporter /usr/local/bin/
```

**Penjelasan**:
- `curl -LO` = download file (L = follow redirect, O = save dengan nama file asli)
- `tar xzf` = extract arsip tar.gz. Flag: `x` extract, `z` gzip, `f` file
- `cp` = copy binary ke /usr/local/bin agar bisa dijalankan dari mana saja
- File `linux-amd64` = binary untuk arsitektur AMD64/x86_64 (sesuai Rocky Linux kita di Windows)

Buat systemd service:
```bash
cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=nobody
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```

```bash
systemctl daemon-reload
systemctl enable --now node_exporter
```

**Penjelasan**: `daemon-reload` = beritahu systemd bahwa ada service file baru. Lalu enable dan start.

#### Verifikasi

```bash
curl http://localhost:9100/metrics | head -20
```

**Penjelasan**: Node Exporter expose metrics di port 9100. `head -20` = tampilkan 20 baris pertama saja.

### 4.7 Diagnosa OOM Killer & Memory

```
📍 Posisi: VM-1
👤 User: root
```

#### Cek Penggunaan Memory

```bash
free -h
```

**Penjelasan**: `free -h` = tampilkan status RAM (total, used, free, available) dalam format human-readable.

#### Cek OOM Killer Log

```bash
dmesg -T | grep -i oom
```

**Penjelasan**:
- `dmesg` = menampilkan log kernel (pesan dari inti OS Linux)
- `-T` = tampilkan timestamp dalam format human-readable
- `| grep -i oom` = filter hanya baris mengandung "oom" (case insensitive)

Jika kernel pernah membunuh proses karena kehabisan memory, akan muncul:
```
[Thu Aug 21 14:30:15 2026] Out of memory: Kill process 10921 (java) score 850
```

#### Cek Proses Paling Boros CPU/Memory

```bash
top -b -n 1 | head -20
```

**Penjelasan**:
- `top` = menampilkan daftar proses real-time
- `-b` = batch mode (output ke terminal, bukan interaktif)
- `-n 1` = tampilkan 1 iterasi saja
- `head -20` = ambil 20 baris pertama

### 4.8 Firewall Management (firewalld)

```
📍 Posisi: VM-3
👤 User: root
```

#### Cek Status Firewall

```bash
systemctl status firewalld
```

#### Cek Zone dan Rules Aktif

```bash
firewall-cmd --list-all
```

**Penjelasan**: Menampilkan semua rules di zone default (services yang diizinkan, port yang terbuka, dll).

#### Tambah Port Custom

```bash
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
```

**Penjelasan**:
- `--add-port=3306/tcp` = buka port 3306 (MySQL) untuk TCP
- `--add-port=9100/tcp` = buka port 9100 (Node Exporter)

#### Verifikasi

```bash
firewall-cmd --list-ports
firewall-cmd --list-services
```

---

## 5. Tahap 4 — Containerization & SOP Escalation

### 5.1 Container Management (Podman — Pengganti Docker di RHEL/Rocky)

```
📍 Posisi: VM-2
👤 User: root
```

> [!NOTE]
> RHEL 9 dan Rocky 9 menggunakan **Podman** sebagai pengganti Docker. Syntax-nya hampir identik!
> `podman` = drop-in replacement untuk `docker`. Perintah `docker ps` → `podman ps`.

#### Install Podman

```bash
dnf install -y podman
```

#### Pull (Download) Container Image

```bash
podman pull docker.io/library/nginx:latest
```

**Penjelasan**: `podman pull` = download container image dari registry. Sama seperti `docker pull`.

#### Jalankan Container

```bash
podman run -d --name web-frontend -p 8080:80 nginx:latest
```

**Penjelasan**:
- `podman run` = buat dan jalankan container baru
- `-d` = **d**etach — jalankan di background
- `--name web-frontend` = beri nama container: `web-frontend`
- `-p 8080:80` = **p**ort mapping — port 8080 di host → port 80 di container
- `nginx:latest` = gunakan image nginx versi terbaru

#### Cek Container Running

```bash
podman ps
```

**Penjelasan**: Menampilkan daftar container yang sedang berjalan (running). Sama seperti `docker ps`.

#### Cek SEMUA Container (termasuk yang mati/exited)

```bash
podman ps -a
```

**Penjelasan**: Flag `-a` = **all** — tampilkan semua container termasuk yang statusnya `Exited`. **Ini penting untuk troubleshooting** — container yang crash tidak muncul di `podman ps` biasa!

Output contoh:
```
CONTAINER ID  IMAGE                  STATUS                     NAMES
a1b2c3d4e5f6  docker.io/nginx:latest Up 5 minutes               web-frontend
b2c3d4e5f6g7  docker.io/nginx:latest Exited (137) 2 minutes ago payment-gateway
```

- `Exited (137)` = container dibunuh oleh OOM Killer (exit code 137 = killed by signal 9 / SIGKILL)
- `Exited (0)` = container berhenti normal
- `Exited (1)` = container error/crash

#### Melihat Log Container

```bash
podman logs --tail 20 web-frontend
```

**Penjelasan**:
- `podman logs` = menampilkan stdout/stderr dari container (log internal aplikasi)
- `--tail 20` = tampilkan 20 baris terakhir saja
- `web-frontend` = nama container

#### Stop dan Hapus Container

```bash
podman stop web-frontend
podman rm web-frontend
```

**Penjelasan**:
- `stop` = hentikan container (mengirim SIGTERM)
- `rm` = hapus container yang sudah di-stop

#### Restart Container yang Crash

```bash
podman start web-frontend
```

### 5.2 Install MariaDB di VM-3

```
📍 Posisi: VM-3
👤 User: root
```

```bash
dnf install -y mariadb-server
systemctl enable --now mariadb
```

```bash
systemctl status mariadb
```

**Verifikasi Port Listening**:
```bash
ss -tulpn | grep 3306
```

**Penjelasan**: Pastikan MariaDB listening di port 3306.

#### Secure Installation

```bash
mysql_secure_installation
```

**Penjelasan**: Wizard interaktif untuk mengamankan instalasi MariaDB — set root password, hapus anonymous user, dll.

### 5.3 SSH Key-Based Authentication (Antar VM)

```
📍 Posisi: VM-1
👤 User: student (bukan root!)
```

> [!IMPORTANT]
> Mulai dari sini kita beralih ke user `student` — karena dalam praktik nyata, **tidak boleh login sebagai root secara langsung** ke server production.

```bash
su - student
```

**Penjelasan**: `su - student` = **s**witch **u**ser ke `student`. Tanda `-` artinya load environment penuh (seperti login baru).

#### Generate SSH Key Pair

```bash
ssh-keygen -t rsa -b 4096
```

**Penjelasan**:
- `ssh-keygen` = generate pasangan kunci SSH (public + private)
- `-t rsa` = tipe algoritma RSA
- `-b 4096` = ukuran key 4096 bit (lebih aman)
- Tekan Enter 3 kali (default file, tanpa passphrase)

#### Kirim Public Key ke VM-2 dan VM-3

```bash
ssh-copy-id student@server2.lab.local
ssh-copy-id student@server3.lab.local
```

**Penjelasan**: `ssh-copy-id` = menyalin public key ke server tujuan. Setelah ini, kamu bisa SSH tanpa password!

#### Test Login Tanpa Password

```bash
ssh student@server2.lab.local hostname
```

**Penjelasan**: SSH ke server2, jalankan command `hostname`, lalu otomatis keluar. Jika berhasil tanpa diminta password = SSH key works!

### 5.4 Sudo Configuration

```
📍 Posisi: VM-1
👤 User: root
```

```bash
visudo
```

**Penjelasan**: `visudo` = editor khusus untuk file `/etc/sudoers` dengan syntax checking. **Selalu gunakan `visudo`**, jangan edit `/etc/sudoers` langsung!

Tambahkan baris:
```
sysadmin  ALL=(ALL)  NOPASSWD: /usr/bin/systemctl status *, /usr/bin/journalctl *
```

**Penjelasan**: User `sysadmin` boleh menjalankan `systemctl status` dan `journalctl` sebagai root tanpa diminta password. Ini membatasi akses — hanya command tertentu yang diizinkan.

### 5.5 Tuned — Performance Tuning

```
📍 Posisi: VM-2
👤 User: root
```

```bash
dnf install -y tuned
systemctl enable --now tuned
```

```bash
tuned-adm list
```

**Penjelasan**: Menampilkan daftar profil tuning yang tersedia (throughput-performance, latency-performance, virtual-guest, dll).

```bash
tuned-adm active
```

**Penjelasan**: Menampilkan profil yang sedang aktif.

```bash
tuned-adm profile throughput-performance
```

**Penjelasan**: Mengaktifkan profil `throughput-performance` — optimal untuk server yang butuh throughput tinggi.

```bash
tuned-adm verify
```

**Penjelasan**: Verifikasi apakah setting sistem sudah sesuai dengan profil aktif.

### 5.6 SOP Escalation — Latihan Menulis Tiket Insiden

```
📍 Posisi: VM-1
👤 User: student
```

Latihan ini mensimulasikan proses eskalasi tiket L1 → L2 sesuai modul assessment.

#### Simulasi: Nginx running tapi backend connection refused

```bash
sudo systemctl status nginx
sudo journalctl -u nginx --no-pager -n 10
sudo ss -tulpn | grep 8080
```

**Penjelasan**: Langkah triage standar L1:
1. Cek status service → **Running** ✅
2. Cek log → ada error `502 connect() failed` ⚠️
3. Cek port backend → port 8080 **tidak listening** ❌

#### Menulis Tiket Eskalasi

```bash
cat > /home/student/escalation-ticket.txt << 'EOF'
=== L1 ESCALATION TICKET ===
Ticket ID    : INC-9912
Priority     : P1 — HIGH
Timestamp    : $(date)
Reported By  : L1 Engineer — student

[TRIAGE RESULTS]
1. systemctl status nginx → Active (running) ✅
2. journalctl -u nginx   → [error] 502 connect() failed (111: Connection refused)
3. ss -tulpn | grep 8080 → No output (port 8080 NOT listening) ❌

[ROOT CAUSE HYPOTHESIS]
Backend Java Application Server di port 8080 tidak berjalan.

[ACTION REQUESTED]
L2 SRE: Periksa dan restart Java/Spring Boot app di port 8080.

[EVIDENCE]
Log snippet dan ss output terlampir di atas.
================================
EOF
```

```bash
cat /home/student/escalation-ticket.txt
```

---

## 6. Latihan Mandiri

## 6. Latihan Sesuai Modul "Training Module RHCSA - OS & Hypervisor Team"

Berikut adalah panduan eksekusi untuk semua **LAB** dan **Task** yang ada di dalam dokumen PDF modul training kamu. Semua command ini dijalankan di **VM-1** kecuali dinyatakan lain.

### 🔹 Domain 1: Essential Tools

#### LAB: Configuring a Collaboration Directory with ACLs
Tujuan: Membuat folder kolaborasi dimana anggota grup bisa baca-tulis, dan file baru otomatis mewarisi grup tersebut.

```bash
# 1. Buat direktori dan grup
mkdir -p /shared/project
groupadd projectteam

# 2. Ubah ownership grup ke projectteam dan set SGID (angka 2 di depan 770)
# SGID (2770) memastikan file yang dibuat di dalamnya otomatis milik grup projectteam
chown :projectteam /shared/project
chmod 2770 /shared/project

# 3. Set ACL spesifik dan Default ACL
setfacl -m g:projectteam:rwx /shared/project
setfacl -d -m g:projectteam:rwx /shared/project
```

#### Task 1: Sticky Bit & Default ACL
Buat `/data/logs`, grup 'ops' bisa write, tapi hanya owner yang bisa hapus (sticky bit + SGID). Default ACL 'auditor' read-only.
```bash
mkdir -p /data/logs
groupadd ops
useradd auditor
chown :ops /data/logs
# 3770 = 3 (Sticky bit=1 + SGID=2), 7 (owner rwx), 7 (group rwx), 0 (others ---)
chmod 3770 /data/logs
setfacl -d -m u:auditor:r-x /data/logs
```

#### Task 2: Archive & Compress
```bash
# Compress dengan xz
tar -cJvf /root/etc-backup.tar.xz /etc/

# Extract untuk verifikasi
mkdir -p /tmp/etc-restore
tar -xJvf /root/etc-backup.tar.xz -C /tmp/etc-restore/
```

#### Task 3: Passwordless SSH
```bash
useradd svcops
su - svcops
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
ssh-copy-id svcops@server2.lab.local
```

### 🔹 Domain 2: Basic Shell Scripting

#### LAB: Disk Health Check Script
```bash
cat > /usr/local/bin/checkdisk.sh << 'EOF'
#!/bin/bash
THRESHOLD=$1
USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')

if [ "$USAGE" -gt "$THRESHOLD" ]; then
    echo "WARNING: Disk usage is ${USAGE}%"
    exit 1
else
    echo "OK: Disk usage is ${USAGE}%"
    exit 0
fi
EOF

chmod +x /usr/local/bin/checkdisk.sh
```

#### Task 4: Create User Script
```bash
cat > /usr/local/bin/newuser.sh << 'EOF'
#!/bin/bash
USERNAME=$1
id "$USERNAME" &>/dev/null
if [ $? -eq 0 ]; then
    echo "User $USERNAME already exists."
else
    useradd "$USERNAME"
    echo "password123" | passwd --stdin "$USERNAME"
    chage -d 0 "$USERNAME"
    echo "User $USERNAME created successfully."
fi
EOF

chmod +x /usr/local/bin/newuser.sh
```

### 🔹 Domain 3: Operating Running Systems

#### Task 5: Boot to Text Mode
```bash
systemctl set-default multi-user.target
```

#### Task 6: Root Password Reset (rd.break)
1. Restart VM, di menu GRUB tekan `e`
2. Cari baris yang dimulai dengan `linux`, tambahkan `rd.break` di akhir baris
3. Tekan `Ctrl+X`
4. Di prompt ketik:
   ```bash
   mount -o remount,rw /sysroot
   chroot /sysroot
   passwd root
   touch /.autorelabel
   exit
   exit
   ```

#### Task 7: Persistent Journal
```bash
mkdir -p /var/log/journal
# Batasi ukuran maksimal 500MB
sed -i 's/#SystemMaxUse=/SystemMaxUse=500M/' /etc/systemd/journald.conf
systemctl restart systemd-journald
```

### 🔹 Domain 4: Local Storage

#### LAB: Provisioning Storage (dari disk /dev/sdb)
```bash
pvcreate /dev/sdb
vgcreate vg_app /dev/sdb
lvcreate -n lv_app -L 3G vg_app
mkfs.xfs /dev/vg_app/lv_app
mkdir -p /app/data
# Ambil UUID
UUID=$(blkid -s UUID -o value /dev/vg_app/lv_app)
echo "UUID=$UUID /app/data xfs defaults 0 0" >> /etc/fstab
mount -a
```

#### Task 8 & 9: Extend VG & LV
```bash
# Task 8
vgcreate vg_backup /dev/sdc
lvcreate -n lv_backup -l 100%FREE vg_backup
mkfs.xfs /dev/vg_backup/lv_backup
mkdir /backup
echo "/dev/vg_backup/lv_backup /backup xfs defaults 0 0" >> /etc/fstab
mount -a

# Task 9 (Extend)
pvcreate /dev/sdd
vgextend vg_data /dev/sdd
lvextend -L +2G /dev/vg_data/lv_app
xfs_growfs /app/data
```

### 🔹 Domain 5: Filesystems, Swap, Automount

#### Task 10: Swap File
```bash
dd if=/dev/zero of=/swapfile bs=1M count=1024
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo "/swapfile none swap defaults 0 0" >> /etc/fstab
```

#### Task 11: Autofs
```bash
dnf install -y autofs
echo "/mnt/nfsdata  /etc/auto.nfsdata" >> /etc/auto.master
echo "data  -rw,sync  10.11.39.21:/export/data" > /etc/auto.nfsdata
systemctl enable --now autofs
```

### 🔹 Domain 6: Deploying Systems & Podman

#### Task 12: Cron Job
```bash
useradd backupsvc
echo "0 2 * * * /usr/local/bin/backup.sh" > /tmp/cron_backup
crontab -u backupsvc /tmp/cron_backup
```

#### LAB & Task 13: Rootless Podman as Systemd Service
```bash
# Login sebagai user webadmin
useradd webadmin
# Set password jika perlu, lalu login SSH atau su -
su - webadmin

# Pull image dan jalankan
podman pull docker.io/library/nginx
podman run -d --name web -p 8080:80 nginx

# Generate systemd file
mkdir -p ~/.config/systemd/user
podman generate systemd --new --name web --files
mv container-web.service ~/.config/systemd/user/

# Enable service
systemctl --user daemon-reload
systemctl --user enable --now container-web

# Keluar dari user webadmin
exit

# Set linger agar service tetap jalan saat user logout (Dijalankan sebagai root)
loginctl enable-linger webadmin
```

#### Task 14: Chrony NTP
```bash
sed -i 's/^pool/#pool/' /etc/chrony.conf
echo "server 10.11.39.21 iburst" >> /etc/chrony.conf
systemctl restart chronyd
timedatectl set-timezone Asia/Jakarta
```

### 🔹 Domain 7 & 8: User Management & Security

#### Task 15: User Aging
```bash
useradd -m -s /bin/bash -G wheel jdoe
echo "password123" | passwd --stdin jdoe
chage -d 0 jdoe          # Force password change on first login
chage -M 60 jdoe         # Expire after 60 days
```

#### Task 16: Sudoers specific commands
```bash
groupadd monitoring
echo "%monitoring ALL=(ALL) NOPASSWD: /usr/bin/systemctl status *, /usr/bin/journalctl" > /etc/sudoers.d/monitoring
```

#### Task 17: SELinux Context Fix
```bash
# Jika apache 403 di /srv/webapp
semanage fcontext -a -t httpd_sys_content_t '/srv/webapp(/.*)?'
restorecon -Rv /srv/webapp
```

#### Task 18: Firewalld
```bash
firewall-cmd --permanent --zone=public --add-port=8443/tcp
firewall-cmd --reload
```

---

## 🎓 FULL EXAM SIMULATION (15 Tasks)

Bagian ini mencakup simulasi full exam di bagian akhir PDF kamu:

1. **Task 1**: `hostnamectl set-hostname l1lab01.lti.local`
2. **Task 2**: `useradd -u 5001 -m -d /home/trainee1 -G wheel trainee1`
3. **Task 3**: `groupadd -g 6001 devops && usermod -aG devops trainee1`
4. **Task 4**: `lvcreate -n lv_web -L 2G vg_data && mkfs.xfs /dev/vg_data/lv_web && mkdir /web && echo "UUID=$(blkid -s UUID -o value /dev/vg_data/lv_web) /web xfs defaults 0 0" >> /etc/fstab && mount -a`
5. **Task 5**: `dd if=/dev/zero of=/swapfile2 bs=1M count=512 && chmod 600 /swapfile2 && mkswap /swapfile2 && swapon /swapfile2 && echo "/swapfile2 none swap defaults 0 0" >> /etc/fstab`
6. **Task 6**: `setenforce 1 && sed -i 's/SELINUX=permissive/SELINUX=enforcing/' /etc/selinux/config && setsebool -P httpd_can_network_connect on`
7. **Task 7**: `firewall-cmd --permanent --zone=public --add-port=9090/tcp && firewall-cmd --reload`
8. **Task 8**: `echo "*/5 * * * * echo healthcheck >> /tmp/hc.log" > /tmp/t1cron && crontab -u trainee1 /tmp/t1cron`
9. **Task 9**: `systemctl set-default multi-user.target`
10. **Task 10**: `tar -czvf /root/etc-$(date +%F).tar.gz /etc/`
11. **Task 11**: `echo "%devops ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/devops`
12. **Task 12**: `setfacl -d -m g:devops:rwx /shared/desdata`
13. **Task 13**: Sama seperti metode Podman Rootless di atas, tapi gunakan user `trainee1`, image `httpd`, dan port `8090`. Jangan lupa `loginctl enable-linger trainee1`.
14. **Task 14**: `chage -M 90 -m 1 -W 7 trainee1`
15. **Task 15**: Update `/etc/chrony.conf` dengan `server 10.11.39.21 iburst`, restart `chronyd`, lalu `timedatectl set-timezone Asia/Jakarta`.

---

## Quick Reference — Peta Command & User

| Command | User | Fungsi | VM |
|---|---|---|---|
| `hostnamectl` | root | Cek/ubah hostname | Semua |
| `ip addr show` | root/student | Lihat IP address | Semua |
| `dnf install -y <pkg>` | root | Install software | Semua |
| `systemctl status/start/stop/enable <svc>` | root | Kelola service | Semua |
| `journalctl -u <svc> -f` | root | Live tail log service | Semua |
| `ss -tulpn` | root | Cek port listening | Semua |
| `df -h` | root/student | Cek ruang disk | Semua |
| `lsblk` | root/student | Lihat disk & partisi | Semua |
| `fdisk /dev/sdb` | root | Partisi disk | VM-1 |
| `pvcreate / vgcreate / lvcreate` | root | Kelola LVM | VM-1 |
| `firewall-cmd` | root | Kelola firewall | Semua |
| `useradd / usermod / passwd` | root | Kelola user | Semua |
| `chmod / chown` | root | Ubah permission/owner | Semua |
| `ls -la / ls -Z` | root/student | Lihat detail file & SELinux | Semua |
| `podman run/ps/logs/stop/rm` | root | Kelola container | VM-2 |
| `curl -I <url>` | root/student | Test HTTP endpoint | Semua |
| `ssh-keygen / ssh-copy-id` | student | Setup SSH key auth | VM-1 |
| `crontab -e / -l` | root | Kelola cron job | Semua |
| `openssl x509` | root | Cek SSL certificate | VM-1 |
| `free -h` | root/student | Cek RAM usage | Semua |
| `top -b -n 1` | root/student | Cek proses CPU/MEM | Semua |
| `dmesg -T \| grep oom` | root | Cek OOM killer kernel | Semua |
| `getenforce / sestatus` | root | Cek status SELinux | Semua |
| `mount / umount` | root | Mount/unmount filesystem | Semua |
| `cat >> /etc/fstab` | root | Tambah mount permanen | Semua |
| `visudo` | root | Edit sudo configuration | Semua |
| `tuned-adm` | root | Performance tuning | VM-2 |

> [!TIP]
> **Aturan Emas:**
> - Command yang mengubah sistem → **harus root** (atau pakai `sudo`)
> - Command yang hanya membaca info → **bisa root atau student**
> - SSH key setup → **sebagai user biasa** (student), bukan root

---

> **Selamat belajar! 🚀** Ikuti step-by-step dari Tahap 1 sampai 4, dan kerjakan Challenge Set di akhir untuk menguji pemahaman.
