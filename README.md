# Lab Debian untuk Belajar IT Infrastruktur

Dokumentasi setup lab Linux pada laptop lama (Lenovo IdeaPad S210, i3-3217U, 4 GB RAM, HDD), dari persiapan flashdisk Ventoy sampai firewall UFW. Ditulis sebagai catatan belajar, lengkap dengan masalah yang ditemui dan cara memperbaikinya.

## Isi

| Bagian | Cakupan |
|---|---|
| [01 - Ventoy dan Instalasi Debian](01-ventoy-instalasi-debian.md) | Spesifikasi, flashdisk Ventoy, persiapan Windows (dual boot), instalasi Debian, pengaturan awal, troubleshooting `sudo` |
| [02 - WiFi dan NetworkManager](02-wifi-networkmanager.md) | Diagnosa WiFi (Atheros AR9565 / `ath9k`), perbaikan WiFi pada instalasi tanpa desktop |
| [03 - SSH dan Firewall (UFW)](03-ssh-ufw.md) | SSH server, SSH key, UFW, daftar kesalahan umum, langkah berikutnya |
| [04 - Hardening SSH dan Nginx](04-hardening-ssh-nginx.md) | Rapikan UFW, hardening SSH, instalasi nginx, panduan hardening nginx |

## Lingkungan

- Laptop: Lenovo IdeaPad S210, UEFI, Secure Boot Off
- OS: Debian stable (netinst), instalasi dengan Xfce atau tanpa desktop
- Klien SSH: Termux di HP Android

## Catatan

- Nama user, alamat IP, dan password pada dokumentasi diganti placeholder (`NAMAUSER`, `IP-LAPTOP`, `PASSWORD`).
- Jangan menyimpan password WiFi, key privat, atau screenshot yang memuatnya di repo ini.
