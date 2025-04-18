# FreeIPA Server & Client Installation Guide (RHEL 9.5)

This guide provides step-by-step instructions for installing a FreeIPA server with integrated DNS on Red Hat Enterprise Linux (RHEL) 9.5, followed by instructions for enrolling RHEL 9.5 clients.

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

### 1.2. Server Prerequisites

#### 1.2.1. Update System
```bash
sudo dnf update -y
sudo reboot  # if kernel/core packages updated
```

#### 1.2.2. Set Hostname (FQDN)
```bash
sudo hostnamectl set-hostname ipa.lab.example.com
hostnamectl status
hostname -f  # should return ipa.lab.example.com
```

#### 1.2.3. Configure /etc/hosts
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

#### 1.2.4. Verify Network Configuration
Ensure static IP, gateway, and DNS (e.g. 8.8.8.8) are properly configured via `nmtui` or `nmcli`.

#### 1.2.5. Verify Repositories
```bash
sudo dnf repolist enabled | grep -E 'baseos|appstream'
```
You should see baseos and appstream repos enabled.

#### 1.2.6. Install and Sync NTP
```bash
sudo dnf install -y chrony
sudo systemctl enable --now chronyd
sleep 60
sudo chronyc sources
```
**Do not proceed** until you see `*` or `+` in the chrony sources output.

#### 1.2.7. Stop Conflicting Services
```bash
sudo systemctl stop dnsmasq
sudo systemctl disable dnsmasq
```
(If using FreeIPA integrated DNS)

### 1.3. Install FreeIPA Server Packages
```bash
sudo dnf install -y ipa-server ipa-server-dns
```
> Skip `ipa-server-dns` if using external DNS.

### 1.4. Run FreeIPA Installation
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

### 1.5. Post-Installation

#### 1.5.1. Verify Firewall Services
```bash
sudo firewall-cmd --list-services
```
Expected: `http https ldap ldaps kerberos kpasswd dns ntp`

To manually add:
```bash
sudo firewall-cmd --permanent --add-service={http,https,ldap,ldaps,kerberos,kpasswd,dns,ntp}
sudo firewall-cmd --reload
```

#### 1.5.2. Set Server to Use Itself as DNS
```bash
IPA_SERVER_IP="192.168.1.202"
CONN_NAME=$(nmcli -g NAME,DEVICE c show --active | grep -v ':lo$' | head -n 1 | cut -d':' -f1)
sudo nmcli con mod "$CONN_NAME" ipv4.dns "$IPA_SERVER_IP"
sudo nmcli con mod "$CONN_NAME" ipv4.ignore-auto-dns yes
sudo nmcli con down "$CONN_NAME" && sudo nmcli con up "$CONN_NAME"
cat /etc/resolv.conf
```
Expected output:
```
search lab.example.com
nameserver 192.168.1.202
```

#### 1.5.3. Authenticate as Admin
```bash
kinit admin
klist
```

#### 1.5.4. Test DNS
```bash
dig @localhost ipa.lab.example.com A +short
dig @localhost -x 192.168.1.202 +short
dig @localhost _ldap._tcp.lab.example.com SRV +short
```

#### 1.5.5. Access the Web UI
Ensure client or management machine resolves `ipa.lab.example.com`. Open browser:
```
https://ipa.lab.example.com
```
Login with `admin` user and the password.

---

## Part 2: FreeIPA Client Installation (RHEL 9.5)

### 2.1. Prerequisites

#### 2.1.1. Update System
```bash
sudo dnf update -y
sudo reboot
```

#### 2.1.2. Configure DNS
```bash
IPA_SERVER_IP="192.168.1.202"
CONN_NAME=$(nmcli -g NAME,DEVICE c show --active | grep -v ':lo$' | head -n 1 | cut -d':' -f1)
sudo nmcli con mod "$CONN_NAME" ipv4.dns "$IPA_SERVER_IP"
sudo nmcli con mod "$CONN_NAME" ipv4.ignore-auto-dns yes
sudo nmcli con down "$CONN_NAME" && sudo nmcli con up "$CONN_NAME"
cat /etc/resolv.conf
```

Verify resolution:
```bash
dig ipa.lab.example.com A +short
dig _ldap._tcp.lab.example.com SRV +short
```

#### 2.1.3. Sync Time
```bash
sudo dnf install -y chrony
sudo systemctl enable --now chronyd
sleep 10
sudo chronyc sources
```

#### 2.1.4. Set Hostname
```bash
sudo hostnamectl set-hostname <client-name>.lab.example.com
```

### 2.2. Install IPA Client
```bash
sudo dnf install -y ipa-client
```

### 2.3. Run IPA Client Install
```bash
sudo ipa-client-install --mkhomedir --enable-dns-updates \
  --server=ipa.lab.example.com \
  --domain=lab.example.com \
  --realm=LAB.EXAMPLE.COM
```
> Use `--force-join` if re-enrolling an existing client.

### 2.4. Post-Install Verification

#### 2.4.1. Verify Access
```bash
ipa user-find admin
```

Create a test user in the IPA Web UI, then on the client:
```bash
su - testuser
pwd   # should show /home/testuser
exit
```

#### 2.4.2. Client Firewall
Clients don’t need incoming port access unless running specific services. Ensure **outbound** access to server’s ports.

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
