# Bagian 4: Hardening SSH dan Web Server Nginx

Seri dokumentasi lab Debian. Cakupan bagian ini: merapikan firewall, hardening SSH, instalasi nginx, dan langkah hardening nginx.

**Status tiap bagian:**

| Bagian | Status |
|---|---|
| 1. Rapikan firewall UFW | Dikerjakan dan terverifikasi di lab |
| 2. Hardening SSH | Dikerjakan dan terverifikasi di lab |
| 3. Instalasi nginx | Dikerjakan dan terverifikasi di lab |
| 4. Hardening nginx | Panduan, **belum dicoba** di lab |

## 1. Rapikan firewall UFW

Aturan SSH sebelumnya tumpang tindih (aturan subnet dan aturan "Anywhere"). Sisakan satu saja.

```bash
sudo ufw status numbered
sudo ufw delete NOMOR        # hapus aturan subnet dan duplikat v6 jika ada
sudo ufw enable
sudo ufw status verbose
```

Hasil yang diharapkan: `Status: active`, default `deny (incoming)` dan `allow (outgoing)`, serta `22/tcp ALLOW IN Anywhere`.

Catatan: setelah `ufw delete` pada aturan IPv4, nomor aturan bergeser. Jalankan `ufw status numbered` lagi sebelum menghapus nomor berikutnya.

## 2. Hardening SSH

### 2.1 Pastikan login dengan key sudah berfungsi (wajib dulu)

Jangan mematikan login password sebelum langkah ini berhasil, karena bisa terkunci. Dari klien (Termux):

```bash
ssh -o PreferredAuthentications=publickey NAMAUSER@IP-LAPTOP
```

Jika langsung masuk tanpa password, key berfungsi. Jika `Permission denied (publickey)`, jalankan `ssh-copy-id NAMAUSER@IP-LAPTOP` dan tes lagi.

### 2.2 Buat konfigurasi hardening

Gunakan file terpisah agar konfigurasi bawaan tidak diubah:

```bash
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
```

Isi:

```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
MaxAuthTries 3
```

### 2.3 Tes sintaks, lalu terapkan

```bash
sudo sshd -t
sudo systemctl reload ssh
```

`sshd -t` yang tidak mengeluarkan pesan apa pun berarti konfigurasi valid. Jika ada error, perbaiki dulu dan jangan reload.

### 2.4 Verifikasi

Pertahankan sesi SSH yang sedang terbuka, lalu uji dari sesi baru:

```bash
# harus berhasil tanpa password
ssh NAMAUSER@IP-LAPTOP

# harus ditolak: Permission denied (publickey)
ssh -o PubkeyAuthentication=no NAMAUSER@IP-LAPTOP
```

Cek konfigurasi aktif di server:

```bash
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin|maxauthtries"
```

### 2.5 Jika terkunci

Akses langsung lewat keyboard laptop. Hapus file konfigurasi lalu reload:

```bash
sudo rm /etc/ssh/sshd_config.d/99-hardening.conf
sudo systemctl reload ssh
```

## 3. Instalasi nginx

```bash
sudo apt install nginx
systemctl status nginx
```

Target: `active (running)`. Buka port 80 di firewall:

```bash
sudo ufw allow 80/tcp
sudo ufw status verbose
```

Tes:

```bash
curl -I http://localhost
```

Harus muncul `HTTP/1.1 200 OK`. Dari HP (WiFi yang sama) buka `http://IP-LAPTOP`. Halaman bawaan atau halaman buatan sendiri tampil.

Mengubah isi halaman bawaan:

```bash
sudo nano /var/www/html/index.nginx-debian.html
```

Untuk file statis, tidak perlu restart nginx.

Membaca log:

```bash
sudo tail -f /var/log/nginx/access.log
journalctl -u nginx --no-pager | tail -20
```

Browser menampilkan tanda "Not secure" untuk alamat `http://`. Itu normal untuk lab lokal tanpa HTTPS.

### Site tambahan di port lain (opsional)

```bash
sudo mkdir -p /var/www/labku
echo "<h1>Lab Debian</h1>" | sudo tee /var/www/labku/index.html
sudo nano /etc/nginx/sites-available/labku
```

Isi:

```
server {
    listen 8080;
    root /var/www/labku;
    index index.html;
}
```

Aktifkan dan terapkan:

```bash
sudo ln -s /etc/nginx/sites-available/labku /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
sudo ufw allow 8080/tcp
```

Polanya sama dengan SSH: **selalu tes sintaks (`nginx -t`) sebelum reload**.

## 4. Hardening nginx (panduan, belum dicoba)

> Langkah di bagian ini belum diuji di lab. Kerjakan satu per satu, jalankan `sudo nginx -t` setiap kali, dan simpan backup sebelum mengedit.

Backup konfigurasi:

```bash
sudo cp -r /etc/nginx /etc/nginx.bak
```

### 4.1 Sembunyikan versi nginx

Secara bawaan nginx menampilkan versinya di header `Server`. Buat file baru:

```bash
sudo nano /etc/nginx/conf.d/hardening.conf
```

Isi:

```
server_tokens off;
```

Terapkan:

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -I http://localhost
```

Header `Server` sekarang hanya `nginx` tanpa nomor versi.

### 4.2 Tambahkan header keamanan dasar

Edit server block yang dipakai (misalnya `/etc/nginx/sites-available/default`) dan tambahkan di dalam blok `server { ... }`:

```
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "no-referrer-when-downgrade" always;
```

Verifikasi:

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -I http://localhost
```

Ketiga header harus tampil di respons.

### 4.3 Blokir akses ke file tersembunyi

Di dalam blok `server { ... }`:

```
location ~ /\. {
    deny all;
}
```

Ini mencegah akses ke file seperti `.git` atau `.env` jika suatu hari ikut terbawa ke folder web.

### 4.4 Batasi metode HTTP (untuk situs statis)

Di dalam blok `server { ... }`:

```
if ($request_method !~ ^(GET|HEAD)$) {
    return 405;
}
```

Hanya cocok untuk situs statis. Jangan dipakai jika nanti ada form atau API.

### 4.5 Batasi laju permintaan (rate limiting)

Di konteks `http` (misalnya di `/etc/nginx/conf.d/hardening.conf`):

```
limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;
```

Lalu di dalam `location /` pada server block:

```
limit_req zone=perip burst=20 nodelay;
```

### 4.6 Periksa hak akses dan proses

```bash
ls -ld /var/www/html /var/www/labku
ps aux | grep nginx
```

Proses worker nginx di Debian berjalan sebagai `www-data`, bukan root. Pastikan folder web tidak dapat ditulis oleh semua orang.

### 4.7 Pantau log

```bash
sudo tail -n 50 /var/log/nginx/access.log
sudo tail -n 50 /var/log/nginx/error.log
```

Perhatikan permintaan ke path aneh (`/wp-login.php`, `/.env`, `/phpmyadmin`) dari IP yang tidak dikenal. Di jaringan lokal biasanya tidak ada, tetapi ini kebiasaan yang berguna.

## 5. Catatan keamanan

- Selama port 80 dan 8080 **tidak di-forward** di router, nginx hanya dapat diakses dari jaringan yang sama. Di WiFi kost, itu berarti semua penghuni yang tersambung bisa membukanya.
- Jangan menaruh data pribadi di folder web.
- HTTPS dengan sertifikat (misalnya Let's Encrypt) baru relevan jika situs diakses dari internet dengan nama domain.
- Simpan token, password, dan key privat di luar repositori ini.

## 6. Ringkasan perintah

| Tujuan | Perintah |
|---|---|
| Tes sintaks SSH | `sudo sshd -t` |
| Terapkan SSH | `sudo systemctl reload ssh` |
| Lihat konfigurasi SSH aktif | `sudo sshd -T` |
| Tes sintaks nginx | `sudo nginx -t` |
| Terapkan nginx | `sudo systemctl reload nginx` |
| Status layanan | `systemctl status nginx` |
| Log akses | `sudo tail -f /var/log/nginx/access.log` |
| Status firewall | `sudo ufw status verbose` |

---

Sebelumnya: [Bagian 3: SSH dan Firewall UFW](03-ssh-ufw.md)
