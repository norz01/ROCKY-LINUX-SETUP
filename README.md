# Panduan Lengkap Latihan Makmal Pentadbiran Pelayan Linux (DFV30122)

Panduan ini menyediakan arahan langkah demi langkah yang terperinci untuk menyelesaikan semua tugasan dalam Latihan Makmal Pentadbiran Pelayan Linux menggunakan **Rocky Linux**. Ia merangkumi penyelesaian untuk ralat-ralat umum yang sering berlaku.

## Anasir Penting Sebelum Bermula

*   **Station Number (SN):** Nombor unik anda yang akan digunakan untuk konfigurasi alamat IP dan nama hos. **Gantikan `<SN>` dalam semua arahan dengan nombor anda** (contoh: jika SN anda ialah `07`, IP pelayan anda ialah `172.16.8.7`).
*   **Akses Root:** Semua arahan memerlukan akses root. Log masuk sebagai `root` atau gunakan `sudo` sebelum setiap arahan.
*   **Editor Teks:** Panduan ini menggunakan `nano`. Jika belum dipasang, jalankan:
    ```bash
    dnf install nano -y
    ```

---

## Pra-Konfigurasi: Sambungan SSH dan Penetapan Nama Hos

Sebelum memulakan tugasan, sambung ke pelayan dan tetapkan nama hos yang betul.

1.  **Sambung melalui SSH:**
    ```bash
    ssh root@UniTechSrv_<SN>
    # Masukkan kata laluan: TrainLab@2025!
    ```

2.  **Tetapkan Nama Hos:**
    Gantikan `<SN>` dengan nombor anda (contoh: `215`).
    ```bash
    hostnamectl set-hostname sysadmin<SN>.unitechlab.net
    ```
    **Contoh:**
    ```bash
    hostnamectl set-hostname sysadmin215.unitechlab.net
    ```
    > Log keluar dan log masuk semula untuk melihat perubahan nama hos pada prompt terminal anda.

---

## Tugasan 1: Pelayan DNS (BIND)

### a) Pemasangan Pakej BIND
```bash
dnf install bind bind-utils -y
```

### b) Konfigurasi Fail Utama BIND (`/etc/named.conf`)
1.  Buka fail konfigurasi:
    ```bash
    nano /etc/named.conf
    ```
2.  Ubah suai bahagian `options` untuk mendengar pada IP pelayan anda dan membenarkan pertanyaan dari rangkaian tempatan.
    ```conf
    options {
        listen-on port 53 { 127.0.0.1; 172.16.8.<SN>; }; // <-- GANTIKAN <SN>
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        // ... (baris lain tidak perlu diubah)
        allow-query     { localhost; 172.16.8.0/24; };
        recursion yes;
        // ...
    };
    ```
3.  Di bahagian paling bawah fail, tambahkan definisi untuk zon hadapan (forward) dan zon undur (reverse).
    ```conf
    // Zon Hadapan untuk unitechlab.net
    zone "unitechlab.net" IN {
        type master;
        file "unitechlab.net.fwd";
        allow-update { none; };
    };

    // Zon Undur untuk 172.16.8.0/24
    zone "8.16.172.in-addr.arpa" IN {
        type master;
        file "unitechlab.net.rev";
        allow-update { none; };
    };
    ```
    Simpan (`Ctrl+O`) dan keluar (`Ctrl+X`).

### c) Cipta Fail Zon Hadapan (Forward Zone)
1.  Buka fail zon hadapan:
    ```bash
    nano /var/named/unitechlab.net.fwd
    ```
2.  Masukkan konfigurasi berikut.
    > **PERHATIAN PENTING:** Pastikan anda menggantikan **SEMUA** kemunculan `<SN>` dengan nombor stesen anda. **JANGAN** salin `<SN>` secara literal. Ini adalah punca ralat `bad name`.
    ```zone
    $TTL 86400
    @   IN  SOA     sysadmin<SN>.unitechlab.net. root.unitechlab.net. (
            2025110501  ; Serial (guna format YYYYMMDDNN)
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
    )
    ; Name Servers
    @   IN  NS      sysadmin<SN>.unitechlab.net.

    ; A Records
    sysadmin<SN>   IN  A       172.16.8.<SN>
    unitechlab.net. IN  A       172.16.8.<SN>
    ```

### d) Cipta Fail Zon Undur (Reverse Zone)
1.  Buka fail zon undur:
    ```bash
    nano /var/named/unitechlab.net.rev
    ```
2.  Masukkan konfigurasi berikut, sekali lagi, **gantikan `<SN>` dengan nombor anda**.
    ```zone
    $TTL 86400
    @   IN  SOA     sysadmin<SN>.unitechlab.net. root.unitechlab.net. (
            2025110501  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
    )
    ; Name Server
    @   IN  NS      sysadmin<SN>.unitechlab.net.

    ; PTR Record
    <SN>    IN  PTR     sysadmin<SN>.unitechlab.net.
    ```

### e) Semak Konfigurasi & Mulakan Servis BIND
1.  Semak sintaks fail konfigurasi:
    ```bash
    named-checkconf
    # (Sepatutnya tiada output jika betul)
    named-checkzone unitechlab.net /var/named/unitechlab.net.fwd
    # (Sepatutnya output OK)
    named-checkzone 8.16.172.in-addr.arpa /var/named/unitechlab.net.rev
    # (Sepatutnya output OK)
    ```
2.  Benarkan servis DNS melalui firewall:
    ```bash
    firewall-cmd --permanent --add-service=dns
    firewall-cmd --reload
    ```
3.  Mulakan dan aktifkan servis BIND:
    ```bash
    systemctl start named
    systemctl enable named
    systemctl status named # Pastikan ia 'active (running)'
    ```

### f) Uji Resolusi DNS
Gunakan `nslookup` untuk mengesahkan.
```bash
nslookup unitechlab.net 127.0.0.1
nslookup 172.16.8.<SN> 127.0.0.1
```
> Ambil tangkapan skrin output kedua-dua arahan ini sebagai bukti.

---

## Tugasan 2: Pelayan DHCP

### a) Pasang Pelayan DHCP```bash
dnf install dhcp-server -y
```

### b) Konfigurasi Pelayan DHCP
1.  Salin fail konfigurasi contoh:
    ```bash
    cp /usr/share/doc/dhcp-server/dhcpd.conf.example /etc/dhcp/dhcpd.conf
    ```
2.  Buka dan ubah suai fail konfigurasi:
    ```bash
    nano /etc/dhcp/dhcpd.conf
    ```
3.  Nyahkomen dan ubah suai blok `subnet` agar sepadan dengan rangkaian anda.
    ```conf
    # A slightly different configuration for an internal subnet.
    subnet 172.16.8.0 netmask 255.255.255.0 {
      range 172.16.8.200 172.16.8.250;
      option domain-name-servers 172.16.8.<SN>; # Gantikan <SN>
      option domain-name "unitechlab.net";
      option routers 172.16.8.1;
      option broadcast-address 172.16.8.255;
      default-lease-time 600;
      max-lease-time 7200;
    }
    ```

### c) Mulakan Servis DHCP
1.  Benarkan servis DHCP melalui firewall:
    ```bash
    firewall-cmd --permanent --add-service=dhcp
    firewall-cmd --reload
    ```
2.  Mulakan dan aktifkan servis:
    ```bash
    systemctl start dhcpd
    systemctl enable dhcpd
    systemctl status dhcpd # Pastikan ia 'active (running)'
    ```

### d) Menunjukkan Bukti DHCP
Anda **MESTI** menggunakan **mesin klien lain** (mesin maya Windows atau Linux) dalam rangkaian yang sama.
1.  Pastikan klien ditetapkan untuk mendapatkan IP secara automatik.
2.  Pada **klien Windows**, buka `Command Prompt` dan jalankan:
    ```cmd
    ipconfig /all
    ```
3.  **Ambil tangkapan skrin** output. Bukti yang berjaya akan menunjukkan:
    *   **IPv4 Address:** Dalam julat `172.16.8.200` - `172.16.8.250`.
    *   **DHCP Server:** Alamat IP pelayan anda (`172.16.8.<SN>`).
    *   **Default Gateway:** `172.16.8.1`.
    *   **DNS Servers:** Alamat IP pelayan anda (`172.16.8.<SN>`).

---

## Tugasan 3: Pelayan FTP (vsftpd)

### a) Pasang vsftpd
```bash
dnf install vsftpd -y
```

### b) Konfigurasi vsftpd
1.  Buka fail konfigurasi:
    ```bash
    nano /etc/vsftpd/vsftpd.conf
    ```
2.  Pastikan tetapan berikut ditetapkan:
    ```conf
    anonymous_enable=NO
    local_enable=YES
    write_enable=YES
    chroot_local_user=YES
    ```
3.  **Penyelesaian Masalah `500 OOPS`:** Tambah baris berikut di hujung fail untuk membenarkan direktori rumah yang boleh ditulis.
    ```conf
    allow_writeable_chroot=YES
    ```

### c) Cipta Pengguna FTP
```bash
useradd ftpuser
passwd ftpuser
# Masukkan kata laluan: P@ssw0rd
```

### d) Mulakan Servis vsftpd
1.  Benarkan servis FTP melalui firewall:
    ```bash
    firewall-cmd --permanent --add-service=ftp
    firewall-cmd --reload
    ```
2.  Tetapkan boolean SELinux:
    ```bash
    setsebool -P ftpd_full_access on
    ```
3.  Mulakan dan aktifkan servis:
    ```bash
    systemctl start vsftpd
    systemctl enable vsftpd
    ```

### e) Sediakan Bukti Log Masuk FTP
Gunakan klien FTP seperti **FileZilla**. Sambung ke `172.16.8.<SN>` dengan nama pengguna `ftpuser` dan kata laluan `P@ssw0rd`. Ambil tangkapan skrin tetingkap FileZilla yang menunjukkan log sambungan yang berjaya.

---

## Tugasan 4: Pelayan Mel (Postfix + Dovecot)

### a) Pasang Postfix dan Dovecot
```bash
dnf install postfix dovecot -y
```

### b) Pasang Klien Mel
> **Penyelesaian Masalah `mail: command not found`:** Pakej `mailx` telah digantikan dengan `s-nail` dalam Rocky Linux terkini.
```bash
dnf install s-nail -y
```

### c) Konfigurasi Postfix (`/etc/postfix/main.cf`)
1.  Buka fail konfigurasi: `nano /etc/postfix/main.cf`
2.  Cari dan ubah suai baris berikut:
    ```conf
    myhostname = sysadmin<SN>.unitechlab.net
    mydomain = unitechlab.net
    myorigin = $mydomain
    inet_interfaces = all
    mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
    home_mailbox = Maildir/
    ```

### d) Konfigurasi Dovecot
1.  Edit `/etc/dovecot/conf.d/10-mail.conf`:
    ```bash
    nano /etc/dovecot/conf.d/10-mail.conf
    ```
    Cari dan tetapkan lokasi mel:
    ```conf
    mail_location = maildir:~/Maildir
    ```
2.  Edit `/etc/dovecot/conf.d/10-auth.conf`:
    ```bash
    nano /etc/dovecot/conf.d/10-auth.conf
    ```
    Nyahkomen dan tetapkan:
    ```conf
    disable_plaintext_auth = no
    ```

### e) Mulakan Servis
1.  Benarkan servis melalui firewall:
    ```bash
    firewall-cmd --permanent --add-service=smtp
    firewall-cmd --permanent --add-service=pop3
    firewall-cmd --permanent --add-service=imap
    firewall-cmd --reload
    ```
2.  Mulakan dan aktifkan kedua-dua servis:
    ```bash
    systemctl start postfix && systemctl enable postfix
    systemctl start dovecot && systemctl enable dovecot
    ```

### f) Tunjukkan Direktori Peti Mel
1.  Cipta seorang pengguna ujian:
    ```bash
    useradd mailuser1
    ```
2.  Hantar e-mel ujian kepadanya:
    ```bash
    echo "Ini adalah e-mel ujian" | mail -s "Ujian" mailuser1
    ```
3.  Sahkan direktori `Maildir` telah dicipta:
    ```bash
    ls -l /home/mailuser1/Maildir/
    ```
    > Ambil tangkapan skrin output yang menunjukkan direktori `new`, `cur`, dan `tmp`.

---

## Tugasan 5 & 6: Pengurusan Pengguna, Kumpulan & Fail

### a) Cipta Kumpulan dan Pengguna
```bash
groupadd TeamLab
useradd -g TeamLab user1
useradd -g TeamLab user2
```

### b) Sahkan Keahlian Kumpulan
```bash
id user1
id user2
```
> Ambil tangkapan skrin output ini.

### c) Cipta Direktori dan Fail Projek
```bash
mkdir /home/TeamProjects
echo "Nama: <Nama Penuh Anda>, ID: <ID Pelajar Anda>" > /home/TeamProjects/readme.txt
```

### d) Tetapkan Kebenaran
```bash
chown :TeamLab /home/TeamProjects
chmod 770 /home/TeamProjects
chmod 660 /home/TeamProjects/readme.txt
```
> Sahkan dengan `ls -ld /home/TeamProjects` dan `ls -l /home/TeamProjects/readme.txt` dan ambil tangkapan skrin.

---

## Tugasan 7: Folder Perkongsian Samba

### a) Pasang Samba
```bash
dnf install samba -y
```

### b) Konfigurasi Samba (`/etc/samba/smb.conf`)
1.  Buka fail konfigurasi: `nano /etc/samba/smb.conf`
2.  Di bahagian paling bawah fail, tambahkan konfigurasi perkongsian berikut.
    > **Penyelesaian Masalah "Access Denied / Minta Kata Laluan"**: Arahan `force user` dan `map to guest` memastikan akses tetamu berfungsi tanpa log masuk.
    ```conf
    [Team_Share]
        path = /home/TeamProjects
        browseable = yes
        writable = yes
        guest ok = yes
        read only = no
        map to guest = Bad User
        force user = nobody
    ```

### c) Kemas Kini Kebenaran Sistem Fail & SELinux
1.  Akses tetamu Samba menggunakan pengguna `nobody`. Kita perlu benarkan pengguna `nobody` untuk menulis ke dalam direktori perkongsian.
    ```bash
    chmod 777 /home/TeamProjects
    ```
2.  Tetapkan konteks SELinux untuk Samba:
    ```bash
    semanage fcontext -a -t samba_share_t "/home/TeamProjects(/.*)?"
    restorecon -Rv /home/TeamProjects
    ```

### d) Mulakan Servis Samba
1.  Benarkan servis Samba melalui firewall:
    ```bash
    firewall-cmd --permanent --add-service=samba
    firewall-cmd --reload
    ```
2.  Mulakan dan aktifkan servis:
    ```bash
    systemctl start smb
    systemctl enable smb
    ```

### e) Tunjukkan Bukti Akses
1.  **Dari Klien Windows:**
    *   Buka File Explorer.
    *   Di bar alamat, taip `\\172.16.8.<SN>\Team_Share`.
    *   Cipta folder baru bernama `Ujian_Windows`.
    *   **Ambil tangkapan skrin** yang menunjukkan anda berjaya mengakses dan mencipta folder.
2.  **Dari Klien Linux:**
    *   Buka Pengurus Fail.
    *   Sambung ke pelayan dengan alamat `smb://172.16.8.<SN>/Team_Share`.
    *   Cipta folder baru bernama `Ujian_Linux`.
    *   **Ambil tangkapan skrin** yang menunjukkan anda berjaya mengakses dan mencipta folder.
