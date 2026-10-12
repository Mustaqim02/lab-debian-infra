# Bagian 2: WiFi dan NetworkManager

Seri dokumentasi lab Debian. Cakupan bagian ini: diagnosa WiFi (chip Atheros AR9565, driver `ath9k`) dan perbaikan WiFi pada instalasi tanpa desktop memakai NetworkManager.

## 1. WiFi: diagnosa dan stabilisasi

Pasang alat diagnosa:
```bash
sudo apt install iw
```

Cek kekuatan sinyal (ganti `wlp2s0` dengan nama interface dari `ip a`):
```bash
sudo iw dev wlp2s0 link
```
Panduan nilai `signal`: -30 s.d. -60 dBm bagus, -60 s.d. -70 lumayan, lebih lemah dari -75 putus-putus. Makin mendekati nol makin kuat (-43 lebih kuat dari -50).

Cek power save dan kestabilan:
```bash
sudo iw dev wlp2s0 get power_save
sudo iw dev wlp2s0 set power_save off
ping -c 30 8.8.8.8
sudo dmesg | grep -i -E "ath9k|wlp2s0|wifi" | tail -20
```

Hasil pada lab ini: sinyal -50 dBm, 0% packet loss, rata-rata 26 ms, log driver bersih. Artinya laptop dan driver sehat. Masalah "harus dekat hotspot" kemungkinan besar dari jaringan WiFi yang ramai.

Jika WiFi memang bermasalah, urutan yang dicoba:
1. Matikan power save permanen:
   ```ini
   # /etc/NetworkManager/conf.d/wifi-powersave.conf
   [connection]
   wifi.powersave = 2
   ```
   lalu `sudo systemctl restart NetworkManager`.
2. Opsi khusus chip AR9565 (hanya jika perlu):
   ```bash
   echo "options ath9k btcoex_enable=1" | sudo tee /etc/modprobe.d/ath9k.conf
   ```
   lalu reboot. Hapus file itu jika malah memburuk.
3. Chip ini hanya **2.4 GHz**. Jika memakai hotspot HP, pilih band 2.4 GHz.
4. Set kode negara: `sudo apt install wireless-regdb`, lalu `sudo iw reg set ID`.
5. Jika semua gagal, curigai hardware (kabel antena longgar). Alternatif: WiFi USB dongle atau USB tethering.

## 2. WiFi pada instalasi tanpa desktop (NetworkManager)

### Gejala
Pada instalasi tanpa desktop, WiFi hanya tersambung ke satu jaringan (yang dipakai saat instalasi). `apt update` hanya jalan saat USB tethering. Di `nmcli device status`, interface WiFi (`wlp2s0`) berstatus `unmanaged` lalu `unavailable`, dan `nmcli device wifi list` kosong.

### Penyebab
Installer menulis konfigurasi WiFi lama di `/etc/network/interfaces` (`allow-hotplug`, `iface ... inet dhcp`, `wpa-ssid`, `wpa-psk`). Konfigurasi ini membuat interface dipegang `ifupdown`, bukan NetworkManager. File ini hanya bisa dibaca dengan `sudo` (tanpa `sudo` terlihat kosong atau `Permission denied`).

### Perbaikan
Kerjakan lewat keyboard laptop (bukan SSH), memakai internet dari USB tethering atau kabel LAN.

1. Pasang NetworkManager:
   ```bash
   sudo apt update
   sudo apt install network-manager
   ```
2. Backup lalu komentari bagian WiFi di `/etc/network/interfaces`:
   ```bash
   sudo cp /etc/network/interfaces /etc/network/interfaces.bak
   sudo nano /etc/network/interfaces
   ```
   Beri `#` pada baris `allow-hotplug wlp2s0`, `iface wlp2s0 inet dhcp`, `wpa-ssid`, dan `wpa-psk`. Biarkan bagian `source` dan `lo` (loopback).
3. Cek `ls /etc/network/interfaces.d/` dan komentari jika ada file yang menyebut interface WiFi.
4. Hentikan `wpa_supplicant` lama dan mulai ulang interface:
   ```bash
   sudo systemctl stop wpa_supplicant
   sudo ip link set wlp2s0 down
   sudo ip link set wlp2s0 up
   sudo systemctl restart NetworkManager
   nmcli device status
   ```
   Target: `wlp2s0` menjadi `disconnected`. Jika belum, `sudo reboot`.
5. Cek blokir radio (opsional): `sudo apt install rfkill`, lalu `sudo rfkill list`. Semua harus `Soft blocked: no` dan `Hard blocked: no`.

### Menyambung dan pindah jaringan
```bash
nmcli device wifi rescan
nmcli device wifi list
sudo nmcli device wifi connect "NAMA-WIFI" password "PASSWORD"
# atau interaktif:
sudo nmtui
```
Jika nama WiFi memiliki spasi di ujung, sambungkan memakai BSSID:
```bash
sudo nmcli device wifi connect AA:BB:CC:DD:EE:FF password "PASSWORD"
```
Cek hasil:
```bash
nmcli device status
hostname -I
ping -c 4 8.8.8.8
sudo apt update
```
NetworkManager menyimpan semua WiFi yang pernah dipakai dan menyambung otomatis. IP akan berbeda di tiap jaringan.

> Keamanan: jangan mengunggah screenshot atau isi file yang memuat password WiFi (`wpa-psk`). Jika sudah terlanjur terlihat, ganti password WiFi tersebut.

---

Sebelumnya: [Bagian 1: Ventoy dan Instalasi](01-ventoy-instalasi-debian.md)  
Lanjut ke: [Bagian 3: SSH dan Firewall UFW](03-ssh-ufw.md)
