# Complete Guide for Linux Server Administration Lab Exercise (DFV30122)

<p align="right">
  <img alt="Language" src="https://img.shields.io/badge/Language-English-blue">&nbsp;
  <a href="./PANDUAN-BM.md"><img alt="Language" src="https://img.shields.io/badge/Language-Bahasa%20Melayu-orange"></a>
</p>

This guide provides detailed, step-by-step instructions to complete all tasks in the Linux Server Administration Lab Exercise using **Rocky Linux**. It includes proactive solutions for common errors encountered during the setup.

## Before You Begin

*   **Station Number (SN):** Your unique number used for IP address and hostname configuration. **Replace `<SN>` in all commands with your number** (e.g., if your SN is `07`, your server IP is `172.16.8.7`).
*   **Root Access:** All commands require root privileges. Log in as `root` or use `sudo` before each command.
*   **Text Editor:** This guide uses `nano`. If it's not installed, run:
    ```bash
    dnf install nano -y
    ```

---

## Pre-Configuration: SSH Connection and Setting the Hostname

Before starting the tasks, connect to the server and set the correct hostname.

1.  **Connect via SSH:**
    ```bash
    ssh root@UniTechSrv_<SN>
    # Enter password: TrainLab@2025!
    ```

2.  **Set the Hostname:**
    Replace `<SN>` with your number (e.g., `215`).
    ```bash
    hostnamectl set-hostname sysadmin<SN>.unitechlab.net
    ```
    **Example:**
    ```bash
    hostnamectl set-hostname sysadmin215.unitechlab.net
    ```
    > Log out and log back in to see the hostname change in your terminal prompt.

---

## Task 1: DNS Server (BIND)

### a) Install BIND Packages
```bash
dnf install bind bind-utils -y
```

### b) Configure the Main BIND File (`/etc/named.conf`)
1.  Open the configuration file:
    ```bash
    nano /etc/named.conf
    ```
2.  Modify the `options` block to listen on your server's IP and allow queries from your local network.
    ```conf
    options {
        listen-on port 53 { 127.0.0.1; 172.16.8.<SN>; }; // <-- REPLACE <SN>
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        // ... (other lines can remain as default)
        allow-query     { localhost; 172.16.8.0/24; };
        recursion yes;
        // ...
    };
    ```
3.  At the very bottom of the file, add the definitions for the forward and reverse zones.
    ```conf
    // Forward Zone for unitechlab.net
    zone "unitechlab.net" IN {
        type master;
        file "unitechlab.net.fwd";
        allow-update { none; };
    };

    // Reverse Zone for 172.16.8.0/24
    zone "8.16.172.in-addr.arpa" IN {
        type master;
        file "unitechlab.net.rev";
        allow-update { none; };
    };
    ```
    Save (`Ctrl+O`) and exit (`Ctrl+X`).

### c) Create the Forward Zone File
1.  Open the forward zone file:
    ```bash
    nano /var/named/unitechlab.net.fwd
    ```
2.  Enter the following configuration.
    > **CRITICAL NOTE:** Make sure you replace **ALL** occurrences of `<SN>` with your station number. **DO NOT** literally copy `<SN>`. This is the cause of the `bad name` error.
    ```zone
    $TTL 86400
    @   IN  SOA     sysadmin<SN>.unitechlab.net. root.unitechlab.net. (
            2025110501  ; Serial (use YYYYMMDDNN format)
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

### d) Create the Reverse Zone File
1.  Open the reverse zone file:
    ```bash
    nano /var/named/unitechlab.net.rev
    ```
2.  Enter the following configuration, again, **replacing `<SN>` with your number**.
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

### e) Check Configuration & Start the BIND Service
1.  Check the syntax of your configuration files:
    ```bash
    named-checkconf
    # (Should have no output if correct)
    named-checkzone unitechlab.net /var/named/unitechlab.net.fwd
    # (Should return an OK status)
    named-checkzone 8.16.172.in-addr.arpa /var/named/unitechlab.net.rev
    # (Should return an OK status)
    ```
2.  Allow the DNS service through the firewall:
    ```bash
    firewall-cmd --permanent --add-service=dns
    firewall-cmd --reload
    ```
3.  Start and enable the BIND service:
    ```bash
    systemctl start named
    systemctl enable named
    systemctl status named # Ensure it is 'active (running)'
    ```

### f) Test DNS Resolution
Use `nslookup` to verify.
```bash
nslookup unitechlab.net 127.0.0.1
nslookup 172.16.8.<SN> 127.0.0.1
```
> Take a screenshot of the output from both commands as proof.

---

## Task 2: DHCP Server

### a) Install the DHCP Server
```bash
dnf install dhcp-server -y
```

### b) Configure the DHCP Server
1.  Copy the example configuration file:
    ```bash
    cp /usr/share/doc/dhcp-server/dhcpd.conf.example /etc/dhcp/dhcpd.conf
    ```
2.  Open and modify the configuration file:
    ```bash
    nano /etc/dhcp/dhcpd.conf
    ```
3.  Uncomment and modify the `subnet` block to match your network.
    ```conf
    # A slightly different configuration for an internal subnet.
    subnet 172.16.8.0 netmask 255.255.255.0 {
      range 172.16.8.200 172.16.8.250;
      option domain-name-servers 172.16.8.<SN>; # Replace <SN>
      option domain-name "unitechlab.net";
      option routers 172.16.8.1;
      option broadcast-address 172.16.8.255;
      default-lease-time 600;
      max-lease-time 7200;
    }
    ```

### c) Start the DHCP Service
1.  Allow the DHCP service through the firewall:
    ```bash
    firewall-cmd --permanent --add-service=dhcp
    firewall-cmd --reload
    ```
2.  Start and enable the service:
    ```bash
    systemctl start dhcpd
    systemctl enable dhcpd
    systemctl status dhcpd # Ensure it is 'active (running)'
    ```

### d) Providing Proof for DHCP
You **MUST** use **another client machine** (a Windows or Linux VM) on the same network.
1.  Ensure the client is set to obtain an IP address automatically.
2.  On the **Windows client**, open `Command Prompt` and run:
    ```cmd
    ipconfig /all
    ```
3.  **Take a screenshot** of the output. Successful proof will show:
    *   **IPv4 Address:** Within the range `172.16.8.200` - `172.16.8.250`.
    *   **DHCP Server:** Your server's IP address (`172.16.8.<SN>`).
    *   **Default Gateway:** `172.16.8.1`.
    *   **DNS Servers:** Your server's IP address (`172.16.8.<SN>`).

---

## Task 3: FTP Server (vsftpd)

### a) Install vsftpd
```bash
dnf install vsftpd -y
```

### b) Configure vsftpd
1.  Open the configuration file:
    ```bash
    nano /etc/vsftpd/vsftpd.conf
    ```
2.  Ensure the following settings are configured:
    ```conf
    anonymous_enable=NO
    local_enable=YES
    write_enable=YES
    chroot_local_user=YES
    ```
3.  **Troubleshooting the `500 OOPS` Error:** Add the following line at the end of the file to allow writable home directories.
    ```conf
    allow_writeable_chroot=YES
    ```

### c) Create the FTP User
```bash
useradd ftpuser
passwd ftpuser
# Enter the password: P@ssw0rd
```

### d) Start the vsftpd Service
1.  Allow the FTP service through the firewall:
    ```bash
    firewall-cmd --permanent --add-service=ftp
    firewall-cmd --reload
    ```
2.  Set the required SELinux boolean:
    ```bash
    setsebool -P ftpd_full_access on
    ```
3.  Start and enable the service:
    ```bash
    systemctl start vsftpd
    systemctl enable vsftpd
    ```

### e) Provide Proof of FTP Login
Use an FTP client like **FileZilla**. Connect to `172.16.8.<SN>` with the username `ftpuser` and password `P@ssw0rd`. Take a screenshot of the FileZilla window showing the successful connection log.

---

## Task 4: Mail Server (Postfix + Dovecot)

### a) Install Postfix and Dovecot
```bash
dnf install postfix dovecot -y
```

### b) Install a Mail Client
> **Troubleshooting `mail: command not found`:** The `mailx` package has been replaced by `s-nail` in modern Rocky Linux.
```bash
dnf install s-nail -y
```

### c) Configure Postfix (`/etc/postfix/main.cf`)
1.  Open the configuration file: `nano /etc/postfix/main.cf`
2.  Find and modify the following lines:
    ```conf
    myhostname = sysadmin<SN>.unitechlab.net
    mydomain = unitechlab.net
    myorigin = $mydomain
    inet_interfaces = all
    mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
    home_mailbox = Maildir/
    ```

### d) Configure Dovecot
1.  Edit `/etc/dovecot/conf.d/10-mail.conf`:
    ```bash
    nano /etc/dovecot/conf.d/10-mail.conf
    ```
    Find and set the mail location:
    ```conf
    mail_location = maildir:~/Maildir
    ```
2.  Edit `/etc/dovecot/conf.d/10-auth.conf`:
    ```bash
    nano /etc/dovecot/conf.d/10-auth.conf
    ```
    Uncomment and set:
    ```conf
    disable_plaintext_auth = no
    ```

### e) Start the Services
1.  Allow the services through the firewall:
    ```bash
    firewall-cmd --permanent --add-service=smtp
    firewall-cmd --permanent --add-service=pop3
    firewall-cmd --permanent --add-service=imap
    firewall-cmd --reload
    ```
2.  Start and enable both services:
    ```bash
    systemctl start postfix && systemctl enable postfix
    systemctl start dovecot && systemctl enable dovecot
    ```

### f) Show the Mailbox Directory
1.  Create a test user:
    ```bash
    useradd mailuser1
    ```
2.  Send a test email to the user:
    ```bash
    echo "This is a test email" | mail -s "Test" mailuser1
    ```
3.  Verify that the `Maildir` directory has been created:
    ```bash
    ls -l /home/mailuser1/Maildir/
    ```
    > Take a screenshot of the output, which should show the `new`, `cur`, and `tmp` directories.

---

## Tasks 5 & 6: User, Group & File Management

### a) Create Group and Users
```bash
groupadd TeamLab
useradd -g TeamLab user1
useradd -g TeamLab user2
```

### b) Verify Group Membership
```bash
id user1
id user2
```
> Take a screenshot of this output.

### c) Create Project Directory and File```bash
mkdir /home/TeamProjects
echo "Name: <Your Full Name>, ID: <Your Student ID>" > /home/TeamProjects/readme.txt
```

### d) Set Permissions
```bash
chown :TeamLab /home/TeamProjects
chmod 770 /home/TeamProjects
chmod 660 /home/TeamProjects/readme.txt
```
> Verify with `ls -ld /home/TeamProjects` and `ls -l /home/TeamProjects/readme.txt` and take a screenshot.

---

## Task 7: Samba Shared Folder

### a) Install Samba
```bash
dnf install samba -y
```

### b) Configure Samba (`/etc/samba/smb.conf`)
1.  Open the configuration file: `nano /etc/samba/smb.conf`
2.  At the very bottom of the file, add the following share configuration.
    > **Troubleshooting "Access Denied / Password Prompt"**: The `force user` and `map to guest` directives ensure guest access works without a login.
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

### c) Update Filesystem & SELinux Permissions
1.  Samba's guest access uses the `nobody` user. We need to allow the `nobody` user to write to the shared directory.
    ```bash
    chmod 777 /home/TeamProjects
    ```
2.  Set the SELinux context for Samba:
    ```bash
    semanage fcontext -a -t samba_share_t "/home/TeamProjects(/.*)?"
    restorecon -Rv /home/TeamProjects
    ```

### d) Start the Samba Service
1.  Allow the Samba service through the firewall:
    ```bash
    firewall-cmd --permanent --add-service=samba
    firewall-cmd --reload
    ```
2.  Start and enable the service:
    ```bash
    systemctl start smb
    systemctl enable smb
    ```

### e) Provide Proof of Access
1.  **From a Windows Client:**
    *   Open File Explorer.
    *   In the address bar, type `\\172.16.8.<SN>\Team_Share`.
    *   Create a new folder named `Windows_Test`.
    *   **Take a screenshot** showing that you have successfully accessed and created the folder.
2.  **From a Linux Client:**
    *   Open the File Manager.
    *   Connect to the server with the address `smb://172.16.8.<SN>/Team_Share`.
    *   Create a new folder named `Linux_Test`.
    *   **Take a screenshot** showing that you have successfully accessed and created the folder.
