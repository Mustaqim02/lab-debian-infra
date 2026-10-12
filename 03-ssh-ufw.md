# Bagian 3: SSH dan Firewall (UFW)

Seri dokumentasi lab Debian. Cakupan bagian ini: SSH server, login dengan SSH key, firewall UFW, daftar kesalahan umum dari seluruh seri, dan langkah berikutnya.

## 1. SSH server dan SSH key

### Cek SSH server
```bash
systemctl status ssh
# jika belum ada:
sudo apt install openssh-server
hostname -I
```

### Masuk dari perangkat lain
Perangkat harus berada di jaringan yang sama.

Dari HP (Termux):
```bash
pkg install openssh
ssh NAMAUSER@IP-LAPTOP
```
Dari Windows 11 (PowerShell/CMD): `ssh NAMAUSER@IP-LAPTOP`

Catatan:
- Pertanyaan fingerprint harus dijawab `yes` lengkap (bukan `y`).
- Jika tidak bisa tersambung padahal pengaturan benar, kemungkinan WiFi memakai *client isolation*. Solusi: hotspot HP atau kabel LAN.
- IP dari DHCP bisa berubah. Cek ulang dengan `hostname -I` jika SSH gagal.

### Login dengan SSH key
Semua dijalankan di **perangkat klien (Termux)**, bukan di dalam sesi SSH ke laptop:
```bash
ssh-keygen -t ed25519
ssh-copy-id NAMAUSER@IP-LAPTOP
ssh NAMAUSER@IP-LAPTOP
```
Password hanya diminta sekali saat `ssh-copy-id`. Jika `ssh-copy-id` tidak tersedia:
```bash
cat ~/.ssh/id_ed25519.pub | ssh NAMAUSER@IP-LAPTOP "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```
Jika masih diminta password, cek izin di laptop: `~/.ssh` harus `700` dan `authorized_keys` harus `600`.

> Peringatan: `ssh-keygen` menimpa key lama jika file sudah ada. Cek `ls ~/.ssh` dulu.

### Peringatan "REMOTE HOST IDENTIFICATION HAS CHANGED"
Muncul setelah Debian diinstal ulang, karena host key SSH laptop berganti sedangkan klien masih mengingat yang lama. Jika Anda sendiri yang menginstal ulang, ini aman. Hapus catatan lama:
```bash
ssh-keygen -R IP-LAPTOP
ssh NAMAUSER@IP-LAPTOP
```
Cocokkan fingerprint baru dengan yang ditampilkan di pesan error, lalu ketik `yes`.

## 2. Firewall dengan UFW

Lakukan lewat SSH atau langsung di laptop. **Urutan penting** agar sesi SSH tidak terkunci: tambahkan aturan SSH sebelum mengaktifkan firewall.

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose
```

Pilih salah satu pola aturan SSH:

| Pola | Perintah | Cocok untuk |
|---|---|---|
| Praktis | `sudo ufw allow 22/tcp` | Sering pindah WiFi |
| Ketat | `sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp` | Jaringan tetap |

Hindari mengaktifkan keduanya sekaligus: aturan "Anywhere" membuat aturan subnet tidak berguna. Pada pola ketat, angka subnet harus cocok dengan jaringan yang dipakai (cek `hostname -I`), dan pindah jaringan membuat SSH terblokir sampai aturan diubah dari keyboard laptop.

Pastikan port 22 **tidak** di-forward ke internet dari router.

Tes: buka sesi SSH kedua sebelum menutup yang pertama. Jika gagal, di sesi pertama jalankan `sudo ufw disable`.

Perintah berguna:
```bash
sudo ufw status numbered
sudo ufw delete NOMOR
sudo ufw disable
sudo ufw reset
```

## 3. Daftar kesalahan umum

| Gejala | Penyebab | Solusi |
|---|---|---|
| `ssh ... Connection timed out` | Salah ketik IP (182.x bukan 192.x) atau IP berubah | Cek `hostname -I` di laptop |
| `iw`/`rfkill`: `command not found` | `/usr/sbin` tidak ada di PATH user biasa | Pakai `sudo` atau tambahkan ke PATH |
| `sudo` ditolak | User tidak ada di grup `sudo` | `su -` lalu `usermod -aG sudo USER` |
| WiFi `unmanaged`/`unavailable` | Konfigurasi `ifupdown` lama | Komentari di `/etc/network/interfaces`, restart NetworkManager |
| Host key berubah | Instalasi ulang | `ssh-keygen -R IP` |
| Masih diminta password SSH | Key belum terpasang | `ssh-copy-id` dari klien |
| `sudo ufw disable` tidak sengaja | Firewall mati | `sudo ufw enable` |

## 4. Langkah berikutnya

- Hardening SSH: matikan login password setelah key berfungsi, larang login root (`PermitRootLogin no`).
- Web server (nginx) untuk latihan layanan, baca log dengan `journalctl`.
- Scripting dan otomasi (bash, cron, Git), lalu Docker dan Ansible.
- Virtualisasi (KVM/Proxmox) dan Kubernetes setelah RAM ditambah.
- Upgrade HDD ke SSD untuk peningkatan performa terbesar.

---

Sebelumnya: [Bagian 2: WiFi dan NetworkManager](02-wifi-networkmanager.md)  
Lanjut ke: [Bagian 4: Hardening SSH dan Nginx](04-hardening-ssh-nginx.md)
