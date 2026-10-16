# Instalasi & Konfigurasi SSH di LXC CentOS (Proxmox)

Rangkuman lengkap dari awal sampai berhasil login SSH.

---

## Tujuan

Menginstall dan mengkonfigurasi SSH Server di container LXC CentOS 100 pada Proxmox VE, sehingga bisa diakses dari komputer client.

---

## Langkah-Langkah

### 1. Install OpenSSH Server

Masuk ke konsol container via Proxmox, lalu jalankan:

```bash
yum install -y openssh-server
```

> Untuk CentOS 8/Stream bisa pakai `dnf install -y openssh-server`

### 2. Generate Host Key (WAJIB)

Container minimal biasanya belum punya host key. Generate dengan:

```bash
ssh-keygen -A
```

Cek hasilnya:

```bash
ls -l /etc/ssh/
```

Pastikan ada `ssh_host_rsa_key`, `ssh_host_ed25519_key`, dll.

### 3. Jalankan Layanan SSH

Karena LXC **tidak punya systemd lengkap**, jangan pakai `systemctl`. Gunakan:

```bash
/usr/sbin/sshd
```

Verifikasi berjalan:

```bash
ps aux | grep sshd
ss -lntp | grep :22
```

Harus muncul `LISTEN ... :22`.

### 4. Set Password Root

Cek status root dulu:

```bash
passwd -S root
```

- Kalau muncul `L` (Locked), unlock:

  ```bash
  passwd -u root
  ```

- Set password baru:

  ```bash
  passwd root
  ```

### 5. Konfigurasi SSH Server

Edit file konfigurasi:

```bash
vi /etc/ssh/sshd_config
```

**Cara edit di `vi`:**

1. Tekan `i` untuk masuk **Insert Mode** (muncul `-- INSERT --`)
2. Hapus tanda `#` pada baris yang perlu diubah
3. Tekan `Esc` untuk keluar dari Insert Mode
4. Ketik `:wq` lalu `Enter` untuk simpan

**Ubah 2 baris ini:**

```conf
PermitRootLogin yes
PasswordAuthentication yes
```

**Atau cara cepat pakai `sed`:**

```bash
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
```

Verifikasi:

```bash
grep -E "PermitRootLogin|PasswordAuthentication" /etc/ssh/sshd_config
```

### 6. Restart Layanan SSH

Setiap kali ubah config, wajib restart:

```bash
pkill sshd
/usr/sbin/sshd
```

### 7. Cek IP Container

```bash
ip addr
```

Catat IP-nya. Kalau container memakai **NAT**, IP yang diakses dari luar adalah **IP Proxmox + port forward**, bukan IP container langsung.

### 8. Test Login dari Client

Dari laptop/PC:

```bash
ssh root@192.168.122.219
```

Kalau berhasil, muncul prompt root container.

---

## Troubleshooting

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| `clear: command not found` | Paket `ncurses` belum terinstall | `yum install -y ncurses` |
| `systemctl: Failed to get D-Bus connection` | LXC tanpa systemd | Pakai `/usr/sbin/sshd` langsung |
| `Permission denied` saat SSH | Root locked / password belum di-set | `passwd -u root && passwd root` |
| `Permission denied` masih muncul | Config SSH belum diubah / belum di-restart | Edit `sshd_config` + `pkill sshd` |
| Tidak bisa edit di vi | Masih di Command Mode | Tekan `i` dulu untuk Insert Mode |

---

## Perintah Debugging Penting

```bash
# Cek log SSH realtime (saat coba login)
tail -f /var/log/secure

# Cek status password root
passwd -S root

# Cek config SSH aktif
grep -E "PermitRootLogin|PasswordAuthentication" /etc/ssh/sshd_config

# Cek SSH jalan atau tidak
ps aux | grep sshd
ss -lntp | grep :22

# Test login verbose dari client
ssh -v root@IP_CONTAINER
```

---

## Catatan Penting

1. **LXC tanpa systemd**: jangan pakai `systemctl`, pakai binary langsung `/usr/sbin/sshd`.
2. **Host key wajib di-generate** dulu dengan `ssh-keygen -A` sebelum start pertama kali.
3. **Config SSH hanya berlaku setelah restart**: `pkill sshd && /usr/sbin/sshd`.
4. **SSH tidak auto-start setelah reboot** karena tidak ada systemd. Untuk auto-start:
   - Buat `/etc/rc.local` berisi `/usr/sbin/sshd`
   - Atau pakai hookscript Proxmox
5. **Keamanan**: setelah berhasil login, sebaiknya:
   - Buat user biasa
   - Matikan kembali `PermitRootLogin`
   - Gunakan SSH key daripada password

---

## Ringkasan Perintah Utama

```bash
# === INSTALASI ===
yum install -y openssh-server
ssh-keygen -A
/usr/sbin/sshd

# === KONFIGURASI ===
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
passwd -u root
passwd root
pkill sshd && /usr/sbin/sshd

# === VERIFIKASI ===
ss -lntp | grep :22
tail -f /var/log/secure

# === LOGIN DARI CLIENT ===
ssh root@192.168.122.219
```

---

**Status: BERHASIL.** SSH Server sudah bisa diakses dari client ke container LXC CentOS.
