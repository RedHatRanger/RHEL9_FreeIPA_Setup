# FreeIPA Server Installation Guide (RHEL 9.5 — Fresh Install)

This guide provides step-by-step instructions for installing a FreeIPA server with integrated DNS on a freshly installed Red Hat Enterprise Linux (RHEL) 9.5 system.

---

## Environment Details
- **RHEL Version**: 9.5  
- **IPA Server FQDN**: `ipa.lab.example.com`  
- **IPA Server IP**: `192.168.1.202`  
- **IPA Domain**: `lab.example.com`  
- **IPA Realm**: `LAB.EXAMPLE.COM`

---

## Part 1: FreeIPA Server Installation (`ipa.lab.example.com`)

### 1.1. Important Considerations
- **FQDN**: Hostname must be `ipa.lab.example.com`.
- **Static IP**: Use `192.168.1.202`.
- **Integrated DNS**: This guide uses FreeIPA's integrated BIND DNS (`--setup-dns`).
  - If using external DNS, skip `--setup-dns` and `ipa-server-dns`, and pre-configure DNS records manually.
- **Resources**: Minimum 4 GB RAM recommended.
- **Time (NTP)**: Ensure accurate time sync using `chronyd`.
- **Firewall**: `firewalld` required. Let the installer configure it automatically.
- **SELinux**: Must be **Enforcing**.
- **Subscription**: System must be registered and subscribed.
- **Passwords**: Prepare strong passwords for:
  - Directory Manager (`cn=Directory Manager`)
  - IPA Admin (`admin`)

### 1.2. Initial System Preparation

#### 1.2.1. Register the System
```bash
sudo subscription-manager register
sudo subscription-manager attach --auto
```

#### 1.2.2. Enable Required Repos
```bash
sudo subscription-manager repos --enable=rhel-9-for-x86_64-baseos-rpms
sudo subscription-manager repos --enable=rhel-9-for-x86_64-appstream-rpms
```

#### 1.2.3. Update All Packages
```bash
sudo dnf update -y
sudo reboot
```

#### 1.2.4. Set Hostname (FQDN)
```bash
sudo hostnamectl set-hostname ipa.lab.example.com
hostnamectl status
hostname -f  # should return ipa.lab.example.com
```

#### 1.2.5. Configure /etc/hosts
```bash
sudo vi /etc/hosts
```
Ensure the following line exists:
```
192.168.1.202    ipa.lab.example.com    ipa
```
Then verify:
```bash
getent hosts $(hostname -f)
```

#### 1.2.6. Configure Static IP and DNS
Ensure the network connection uses a static IP and temporary DNS (like `8.8.8.8`) via `nmtui` or `nmcli`.

#### 1.2.7. Set SELinux to Enforcing
```bash
sudo setenforce 1
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
```

#### 1.2.8. Install Required Packages
```bash
sudo dnf install -y firewalld chrony ipa-server ipa-server-dns
```

#### 1.2.9. Enable and Start Services
```bash
sudo systemctl enable --now firewalld
sudo systemctl enable --now chronyd
```

#### 1.2.10. Sync NTP Time
```bash
sleep 60
sudo chronyc sources
```
**Do not proceed** until you see `*` or `+` in the chrony sources output.

#### 1.2.11. Stop Conflicting Services
```bash
sudo systemctl stop dnsmasq
sudo systemctl disable dnsmasq
```
(If using FreeIPA integrated DNS)

---

### 1.3. Run FreeIPA Installation
```bash
sudo ipa-server-install --setup-dns
```
Follow the prompts:
- **Host Name**: `ipa.lab.example.com`
- **Domain**: `lab.example.com`
- **Realm**: `LAB.EXAMPLE.COM`
- **Passwords**: Enter strong passwords for Directory Manager and IPA Admin
- **DNS Forwarders**: e.g. `8.8.8.8 1.1.1.1`
- **Reverse Zone**: Accept default or configure accordingly
- Confirm configuration and proceed
- Allow installer to configure firewall and chrony (recommended)

---

### 1.4. Post-Installation

#### 1.4.1. Verify Firewall Services
```bash
sudo firewall-cmd --list-services
```
Expected: `http https ldap ldaps kerberos kpasswd dns ntp`

To manually add:
```bash
sudo firewall-cmd --permanent --add-service={http,https,ldap,ldaps,kerberos,kpasswd,dns,ntp}
sudo firewall-cmd --reload
```

#### 1.4.2. Set Server to Use Itself as DNS
Use `nmtui` to configure your active network interface to use the FreeIPA server as its DNS resolver:

1. Run the text-based interface:
   ```bash
   sudo nmtui
   ```
2. Select **Edit a connection**.
3. Choose your active network connection (e.g., `Wired connection 1`) and hit **Enter**.
4. In the IPv4 CONFIGURATION section:
   - Change **Method** to `Manual`
   - Enter your static IP address (e.g., `192.168.1.202/24`), gateway, and DNS (e.g., `192.168.1.202`)
   - Ensure **Automatic DNS** is disabled
5. Save and exit back to the main menu.
6. Choose **Activate a connection** → Restart the interface.
7. Exit `nmtui`

Verify that `/etc/resolv.conf` points to your own IP:
```bash
cat /etc/resolv.conf
```
Expected output:
```
search lab.example.com
nameserver 192.168.1.202
```
```
Expected output:
```
search lab.example.com
nameserver 192.168.1.202
```

#### 1.4.3. Authenticate as Admin
```bash
kinit admin
klist
```

#### 1.4.4. Test DNS
```bash
dig @localhost ipa.lab.example.com A +short
dig @localhost -x 192.168.1.202 +short
dig @localhost _ldap._tcp.lab.example.com SRV +short
```

#### 1.4.5. Access the Web UI
Ensure client or management machine resolves `ipa.lab.example.com`. Open browser:
```
https://ipa.lab.example.com
```
Login with `admin` user and the password.

---

## Appendix: Modify Windows Hosts File
Use this if a Windows machine can't resolve the IPA server and isn’t using it for DNS.

1. Open Notepad as Administrator
2. Open file: `C:\Windows\System32\drivers\etc\hosts`
3. Add:
```
192.168.1.202    ipa.lab.example.com
```
4. Save and exit
5. Run:
```cmd
ipconfig /flushdns
```

---

## Disclaimer
Test in non-production environments. Backup all data before changes. Refer to [official Red Hat IdM documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/installing_identity_management/index) for advanced use cases.
