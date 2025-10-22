# ROCKY-LINUX-SETUP

Baiklah, faham. Anda mahukan satu dokumen rujukan lengkap—sebuah *master playbook*—yang mengandungi setiap arahan dan setiap blok konfigurasi yang diperlukan dari awal hingga akhir, supaya anda tidak perlu merujuk ke mana-mana tempat lain. Ini adalah amalan dokumentasi yang sangat baik.

Mari kita bina panduan definitif tersebut. Gantikan semua nilai `SN` dan `IP` dengan spesifikasi anda.

---

### **Pelan Tindakan Definitif & Lengkap: Konfigurasi Pelayan Rocky Linux**

**Spesifikasi Asas yang Digunakan dalam Panduan Ini:**
*   **Hostname:** `GRP-2020`
*   **Domain:** `grpcloud.com`
*   **Alamat IP Pelayan:** `172.16.8.15` (Gantikan `enp0s3` dengan peranti anda)
*   **Subnet:** `172.16.8.0/24`
*   **Gateway:** `172.16.8.1` (Anggapan)
*   **Direktori Web:** `/web_grpcloud`

---

#### **Fasa 1: Asas Rangkaian & Pemasangan Pakej**

**1. Konfigurasi Rangkaian Awal (dengan DNS Awam Sementara)**
```bash
# Gantikan nilai ini jika perlu
IP_ADDR="172.16.8.15"
PREFIX="24"
GATEWAY="172.16.8.1"
DEVICE="enp0s3"

nmcli con mod $DEVICE ipv4.addresses $IP_ADDR/$PREFIX
nmcli con mod $DEVICE ipv4.gateway $GATEWAY
nmcli con mod $DEVICE ipv4.dns "8.8.8.8 1.1.1.1"
nmcli con mod $DEVICE ipv4.method manual
nmcli con up $DEVICE
```

**2. Kemas Kini Sistem & Pemasangan Semua Pakej**
```bash
dnf update -y
dnf install -y httpd samba vsftpd cups kea bind bind-utils policycoreutils-python-utils
```

**3. Tetapkan Hostname dan Fail `/etc/hosts`**
```bash
hostnamectl set-hostname GRP-2020
echo "172.16.8.15 GRP-2020.grpcloud.com GRP-2020" >> /etc/hosts
```

---

#### **Fasa 2: Konfigurasi Perkhidmatan & Firewall**

**4. Konfigurasi Apache (Web Server)**
```bash
# Cipta direktori dan fail contoh
mkdir -p /web_grpcloud
echo "<h1>Welcome to grpcloud.com</h1>" > /web_grpcloud/index.html
chown -R apache:apache /web_grpcloud

# Cipta fail konfigurasi VirtualHost
cat <<EOF > /etc/httpd/conf.d/grpcloud.conf
<VirtualHost *:80>
    ServerName www.grpcloud.com
    ServerAlias grpcloud.com
    DocumentRoot /web_grpcloud
    <Directory /web_grpcloud>
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
EOF

# Mulakan perkhidmatan dan buka firewall
systemctl enable --now httpd
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

**5. Konfigurasi Samba, FTP, dan CUPS**
```bash
# Konfigurasi Samba (konfigurasi minimum untuk perkongsian direktori home)
# Pastikan blok [homes] wujud dalam /etc/samba/smb.conf
systemctl enable --now smb
firewall-cmd --permanent --add-service=samba

# Konfigurasi FTP (vsftpd) untuk pengguna tempatan
sed -i 's/anonymous_enable=YES/anonymous_enable=NO/' /etc/vsftpd/vsftpd.conf
echo "local_enable=YES" >> /etc/vsftpd/vsftpd.conf
echo "write_enable=YES" >> /etc/vsftpd/vsftpd.conf
echo "chroot_local_user=YES" >> /etc/vsftpd/vsftpd.conf
echo "allow_writeable_chroot=YES" >> /etc/vsftpd/vsftpd.conf
systemctl enable --now vsftpd
firewall-cmd --permanent --add-service=ftp

# Konfigurasi CUPS (Print Server)
systemctl enable --now cups
firewall-cmd --permanent --add-service=ipp

# Muat semula firewall untuk semua perubahan
firewall-cmd --reload
```

---

#### **Fasa 3: Konfigurasi Perkhidmatan Kompleks (DHCP & DNS)**

**6. Konfigurasi DHCP Server (Kea)**
```bash
# Cipta fail /etc/kea/kea-dhcp4.conf dengan kandungan lengkap
cat <<EOF > /etc/kea/kea-dhcp4.conf
{
"Dhcp4": {
    "interfaces-config": {
        "interfaces": [ "enp0s3" ]
    },
    "lease-database": {
        "type": "memfile",
        "persist": true,
        "name": "/var/lib/kea/dhcp4.leases"
    },
    "subnet4": [
        {
            "subnet": "172.16.8.0/24",
            "id": 1,
            "pools": [ { "pool": "172.16.8.20 - 172.16.8.30" } ],
            "option-data": [
                { "name": "domain-name-servers", "data": "172.16.8.15" },
                { "name": "domain-name", "data": "grpcloud.com" },
                { "name": "routers", "data": "172.16.8.1" }
            ]
        }
    ]
}
}
EOF

# Uji fail konfigurasi
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf

# Jika ujian berjaya, mulakan perkhidmatan dan buka firewall
systemctl enable --now kea-dhcp4
firewall-cmd --permanent --add-service=dhcp
firewall-cmd --reload```
```
**7. Konfigurasi BIND (DNS Server) - Termasuk Fail Zon**
```bash
# 1. Edit fail konfigurasi utama /etc/named.conf
sed -i 's/listen-on port 53 { 127.0.0.1; };/listen-on port 53 { 127.0.0.1; 172.16.8.15; };/' /etc/named.conf
sed -i 's/allow-query     { localhost; };/allow-query     { localhost; 172.16.8.0\/24; };/' /etc/named.conf

# 2. Tambah definisi zon pada akhir /etc/named.conf
cat <<EOF >> /etc/named.conf

zone "grpcloud.com" IN {
    type master;
    file "grpcloud.com.zone";
    allow-update { none; };
};
EOF

# 3. Cipta fail zon (forward lookup)
cat <<EOF > /var/named/grpcloud.com.zone
\$TTL 86400
@   IN  SOA     GRP-2020.grpcloud.com. root.grpcloud.com. (
        $(date +%Y%m%d%S)   ; Serial (dinamik)
        3600                ; Refresh
        1800                ; Retry
        604800              ; Expire
        86400 )             ; Minimum TTL
;
@           IN  NS      GRP-2020.grpcloud.com.
GRP-2020    IN  A       172.16.8.15
www         IN  A       172.16.8.15
EOF

# 4. Tetapkan pemilikan yang betul pada fail zon
chown root:named /var/named/grpcloud.com.zone

# 5. Uji sintaks fail konfigurasi
named-checkconf

# 6. Uji sintaks fail zon
named-checkzone grpcloud.com /var/named/grpcloud.com.zone

# 7. Jika ujian berjaya, mulakan perkhidmatan dan buka firewall
systemctl enable --now named
firewall-cmd --permanent --add-service=dns
firewall-cmd --reload
```

**8. Kemas Kini Rangkaian Akhir (Guna DNS Sendiri)**
```bash
nmcli con mod $DEVICE ipv4.dns "172.16.8.15"
nmcli con up $DEVICE
```

---

#### **Fasa 4: Pengurusan Pengguna dan Fail**

**9. Cipta Pengguna dan Kumpulan**
```bash
# Senarai pengguna dan gelung untuk mencipta akaun & set kata laluan
USERS="ahmad aslam marissa misya norman khairul aming alice leeyana jarjit"
for user in $USERS; do
    useradd $user
    echo 'xyz@123' | passwd --stdin $user
done

# Cipta semua kumpulan
groupadd GRPProj1; groupadd GRPProj2; groupadd GRPProj3; groupadd GRPProj2020
```

**10. Ubah Suai Keahlian Kumpulan**
```bash
usermod -aG GRPProj1 ahmad; usermod -aG GRPProj1 aslam; usermod -aG GRPProj1 marissa
usermod -aG GRPProj2 misya; usermod -aG GRPProj2 norman; usermod -aG GRPProj2 khairul
usermod -aG GRPProj3 alice; usermod -aG GRPProj3 leeyana; usermod -aG GRPProj3 jarjit
for user in $USERS; do usermod -aG GRPProj2020 $user; done```

**11. Pengurusan Fail dan Direktori dengan ACL**
```bash
mkdir /home/GRPProj2020
echo -e "Ahmad\nNorman" > /home/GRPProj2020/readme.txt
chown root:root /home/GRPProj2020
chmod 770 /home/GRPProj2020
setfacl -m u:ahmad:rwx /home/GRPProj2020
setfacl -m u:aslam:rwx /home/GRPProj2020
```

---

#### **Fasa 5: Pengesahan Akhir**

**12. Uji Semua Perkhidmatan**
```bash
# Uji Web Server dari klien: Buka http://172.16.8.15 dalam pelayar
# Uji DNS
dig www.grpcloud.com @172.16.8.15
# Uji DHCP: Konfigurasi klien untuk guna DHCP dan sahkan ia mendapat IP dari julat yang betul
# Uji FTP: cuba log masuk dengan ftp://admin.grpcloud@172.16.8.15
# Uji komunikasi VM
ping <IP_GRP-Lab1>
ping <IP_GRP-Lab2>
```Ini adalah *master playbook* anda. Setiap langkah, arahan, dan kandungan fail yang diperlukan kini berada dalam satu dokumen. Simpan ini dengan baik. Ia adalah bukti kerja dan pengetahuan anda.
ping <IP_GRP-Lab2>
```Ini adalah *master playbook* anda. Setiap langkah, arahan, dan kandungan fail yang diperlukan kini berada dalam satu dokumen. Simpan ini dengan baik. Ia adalah bukti kerja dan pengetahuan anda.
