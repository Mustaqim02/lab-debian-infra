# Bagian 1: Persiapan Ventoy dan Instalasi Debian

Seri dokumentasi lab Debian untuk belajar IT infrastruktur pada laptop lama (Lenovo IdeaPad S210, i3-3217U, 4 GB RAM, HDD).
Cakupan bagian ini: spesifikasi, flashdisk Ventoy, instalasi Debian, pengaturan awal, dan troubleshooting `sudo`.

## 1. Spesifikasi dan hasil pengecekan awal

| Item | Nilai |
|---|---|
| Laptop | Lenovo IdeaPad S210 (BIOS 2013) |
| CPU | Intel Core i3-3217U (2 core / 4 thread) |
| RAM | 4 GB |
| Penyimpanan | HDD |
| WiFi | Qualcomm Atheros AR956x (AR9565), driver `ath9k` |
| Mode boot | UEFI, Secure Boot **Off** |
| OS awal | Windows 11 Pro 23H2 |

Cara mengecek di Windows:
- Mode BIOS dan Secure Boot: `Win + R`, ketik `msinfo32`
- Chip WiFi: Device Manager, Network adapters
- Virtualisasi: Task Manager, Performance, CPU

Catatan: spek ini cukup untuk Linux CLI, jaringan, layanan dasar, dan Docker ringan.
Yang terbatas adalah RAM untuk banyak VM atau Kubernetes penuh.

## 2. Persiapan flashdisk Ventoy

Flashdisk sudah disetup Ventoy dengan dua partisi:

1. `VTOYEFI` (FAT): partisi boot Ventoy, **jangan diubah**
2. `Ventoy` (exFAT): tempat menyimpan file ISO

Langkah:
1. Unduh ISO **Debian stable netinst (amd64)** dari debian.org.
2. Salin file `.iso` ke partisi **Ventoy** (exFAT), di samping ISO lain (misalnya Windows).
3. Boot dari flashdisk (tombol Novo atau F12 pada IdeaPad), lalu pilih ISO Debian di menu Ventoy.
4. Pilih **Normal mode**.

Tips:
- Karena Secure Boot Off, tidak perlu langkah enroll key Ventoy.
- Jika ISO tidak mau boot, update Ventoy lewat Ventoy2Disk (pilih **Update**, bukan Install, agar ISO tidak terhapus).
- Jika layar hitam atau berhenti, tekan `Ctrl+R` di menu Ventoy untuk **Grub2 mode**.
- Jika installer tidak menemukan media instalasi, pilih *Back*, lalu *Detect and mount CD-ROM* lagi.
- Siapkan kabel LAN atau USB tethering dari HP sebagai cadangan jika WiFi belum terdeteksi installer.

## 3. Persiapan Windows (jika dual boot)

1. Backup data penting.
2. Matikan BitLocker (cek dengan `manage-bde -status` di CMD admin).
3. Matikan Fast Startup dan hibernate: `powercfg /h off` (CMD admin).
4. Jalankan Optimize Drives (defrag) pada HDD.
5. Disk Management, klik kanan drive C, **Shrink Volume**, sisakan sekitar 40-60 GB (contoh 51200 MB). Biarkan sebagai *Unallocated*.

## 4. Instalasi Debian

1. Boot ISO Debian dari Ventoy, pilih **Install**.
2. Atur bahasa, lokasi, keyboard, dan hostname. Buat user biasa.
3. Partisi:
   - Untuk dual boot UEFI: pakai ulang partisi EFI yang ada sebagai `/boot/efi` dan **jangan diformat**.
   - Boleh pakai *Guided: use the largest continuous free space*.
   - Pastikan memilih **HDD internal**, bukan flashdisk.
4. Software selection:
   - Desktop ringan **Xfce**, atau tanpa desktop sama sekali (untuk server/lab).
   - Centang **SSH server** dan **standard system utilities**.
5. Pasang GRUB di disk utama.

## 5. Pengaturan awal setelah instalasi

```bash
sudo apt update && sudo apt full-upgrade
uname -a
free -h
lsblk
ip a
```

Jika Windows tidak muncul di menu GRUB (dual boot):
```bash
sudo os-prober
sudo update-grub
```
Jika `os-prober` tidak menemukan apa pun, set `GRUB_DISABLE_OS_PROBER=false` di `/etc/default/grub`, lalu `sudo update-grub` lagi.

Optimasi ringan untuk RAM 4 GB dan HDD:
```bash
sudo apt install zram-tools tlp
```

Opsional: opsi `noatime` pada `/etc/fstab` (untungnya kecil, Debian sudah memakai `relatime`).
Selalu backup dulu sebelum edit:
```bash
sudo cp /etc/fstab /etc/fstab.bak
sudo nano /etc/fstab
# tambahkan ,noatime pada opsi partisi / (ext4), tanpa spasi
sudo mount -o remount /
mount | grep " / "
```
Jika ada masalah: `sudo cp /etc/fstab.bak /etc/fstab`.

## 6. Troubleshooting sudo

`sudo` meminta password **user sendiri**, bukan password root. Saat mengetik, layar tidak menampilkan apa pun.

Jika muncul `user is not in the sudoers file` (biasanya karena password root diisi saat instalasi):
```bash
su -
usermod -aG sudo NAMAUSER
exit
```
Lalu logout dan login lagi. Cek dengan `groups` (harus ada `sudo`).

Catatan: perintah di `/usr/sbin` (misalnya `iw`, `rfkill`, `ufw`) kadang `command not found` untuk user biasa karena `/usr/sbin` tidak ada di PATH. Solusi: pakai `sudo`, tulis path lengkap, atau tambahkan ke PATH:
```bash
echo 'export PATH=$PATH:/usr/sbin:/sbin' >> ~/.bashrc
source ~/.bashrc
```

---

Lanjut ke: [Bagian 2: WiFi dan NetworkManager](02-wifi-networkmanager.md)
