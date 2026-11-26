# Maintenance Server — Armbian STB S905x

Repositori ini berisi dokumentasi dan file konfigurasi untuk server rumah berbasis STB Amlogic S905x yang menjalankan Armbian. Tujuannya adalah menyimpan *state* konfigurasi yang sudah di-*hardening* agar mudah direplikasi, di-*rollback*, atau diaudit kembali.

## 🖥️ Spesifikasi Perangkat
- **Perangkat**: STB Amlogic S905x (RAM 1.75 GB)
- **OS**: Armbian 26.11.0 (Ubuntu Noble)
- **Kernel**: 6.12.117-ophub
- **Penyimpanan**: eMMC 6 GB (root) + microSD 15 GB + USB 29 GB
- **Jaringan**: LAN `192.168.1.10/24` (NAT, tidak ada IP publik)

## 🟢 Layanan yang Berjalan
| Layanan | Port | Status |
|---|---|---|
| SSH | 22/tcp | Aktif, root login dibatasi |
| Nginx | 80/tcp | Aktif, menyajikan `/mnt/usb/www/welcome` |
| Fail2ban | — | Aktif, jail `sshd` |
| UFW | — | Aktif, default deny incoming |
| Cloudflared | UDP 17760, 19972 | Aktif |
| Chrony | — | Sinkronisasi NTP |
| vnStat | — | Monitoring trafik |
| ZRAM | 896 MB | Swap terkompresi |

## 🔐 Konfigurasi Keamanan Kunci
- **SSH**: `PermitRootLogin prohibit-password`, `PasswordAuthentication` masih aktif (risiko rendah karena NAT), `MaxAuthTries 3`, `X11Forwarding no`.
- **PAM**: `faillock` dengan `deny=5`, `unlock_time=900`, `even_deny_root=true`.
- **Sysctl**: ICMP redirect dimatikan, `rp_filter` longgar, IP forwarding aktif (untuk Cloudflared/VPN).
- **Audit**: Skor BOB ≤8/10; Access Control 6/10. Sisa temuan: passphrase kunci privat, `su` tanpa batasan, `pam_pwquality`, SUID `mount.nfs`, dan `/tmp` tanpa `nodev`.

## ⚙️ Optimasi Performa
- **CPU Governor**: `schedutil`, batas 500–1512 MHz, diterapkan oleh `set-cpu-freq.service` (persisten).
- **ZRAM**: 896 MB, algoritma `lzo`, prioritas swap 5.
- **Journald**: Dikonfigurasi `Storage=persistent`; `systemd-journal-flush` diberi `ExecStartPre=/bin/sleep 3` agar menunggu `armbian-ramlog`.

## 🛠️ Perintah Maintenance
```bash
# Update paket
sudo apt update && sudo apt upgrade -y

# Cek status layanan
sudo systemctl status ssh nginx fail2ban ufw

# Audit keamanan
sudo bob

# Cek penggunaan memori & swap
free -h
swapon --show

# Cek suhu & frekuensi CPU
armbianmonitor -m

# Backup konfigurasi (contoh)
tar -czf /mnt/usb/backup-config-$(date +%F).tar.gz /etc/ssh /etc/systemd /etc/sysctl.d /etc/security
