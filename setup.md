https://www.youtube.com/watch?v=ueU-Ni0_0wQ

Okay, I've updated the Markdown template with your specific details: 192.168.1.202 for the IP address and ipa.lab.example.com for the FQDN.

# FreeIPA Server Installation on RHEL 9.5

This guide provides step-by-step instructions for installing a FreeIPA server with integrated DNS on a fresh Red Hat Enterprise Linux (RHEL) 9.5 system.

**Author:** Your Name/AI Assistant
**Date:** 2023-10-27 (Update as needed)
**RHEL Version:** 9.5
**Target Server FQDN:** `ipa.lab.example.com`
**Target Server IP:** `192.168.1.202`
**Target Domain:** `lab.example.com`
**Target Realm:** `LAB.EXAMPLE.COM`

---

## 1. Important Considerations Before You Start

*   **FQDN is CRUCIAL:** Your server **MUST** have a Fully Qualified Domain Name (FQDN) configured as its hostname (`ipa.lab.example.com`).
*   **Static IP Address:** The FreeIPA server requires a static IP address (`192.168.1.202`). DHCP is not suitable for the server itself.
*   **DNS Strategy:** This guide assumes you will use FreeIPA's integrated BIND DNS server (`--setup-dns`). If using external DNS, manual configuration of SRV and other records is required *before* installation, and you would omit `--setup-dns` and the `ipa-server-dns` package.
*   **Resources:** Allocate sufficient resources. Red Hat recommends **at least 4 GB of RAM** (more is better, especially for larger environments or replicas). CPU requirements depend on load.
*   **Time Synchronization (NTP):** Accurate time is **ESSENTIAL** for Kerberos. Ensure `chronyd` is installed, enabled, running, and synchronized *before* starting the IPA installation.
*   **Firewall (`firewalld`):** FreeIPA requires numerous ports. The installer can configure `firewalld` automatically (recommended). This guide includes manual steps for verification or if the automatic step fails.
*   **SELinux:** Keep SELinux in `Enforcing` mode. The installer handles necessary policies. Do **NOT** disable SELinux.
*   **RHEL Subscription:** Ensure the system is registered (`subscription-manager register`) and subscribed to the BaseOS and AppStream repositories.
*   **Passwords:** Prepare strong passwords for the Directory Manager (`cn=Directory Manager`) and the initial IPA admin user (`admin`). Store these securely!
*   **Domain/Realm Names:** Using Domain: `lab.example.com`, Realm: `LAB.EXAMPLE.COM`.

---

## 2. Prerequisites

Perform these steps on the target RHEL 9.5 server.

### 2.1. Update System
```bash
sudo dnf update -y
# Reboot is recommended after kernel or other core updates
sudo reboot

2.2. Set Static Hostname (FQDN)
sudo hostnamectl set-hostname ipa.lab.example.com

# Verify
hostnamectl status
hostname -f
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

The output of hostname -f must be ipa.lab.example.com.

2.3. Configure /etc/hosts

Ensure the server's FQDN resolves locally to its static IP before DNS is fully configured.

# Edit the hosts file
sudo vi /etc/hosts

# Add a line like this AFTER the 127.0.0.1 and ::1 lines:
# <Static-IP>      <FQDN>                <Short-Hostname>
192.168.1.202    ipa.lab.example.com   ipa

# Save and close the file (:wq in vi)

# Verify resolution (should show the static IP)
getent hosts $(hostname -f)
# Expected output: 192.168.1.202    ipa.lab.example.com
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
2.4. Verify Network Configuration

Ensure the static IP address 192.168.1.202, netmask, gateway, and temporary external DNS (like your router or 8.8.8.8) are configured. You can use nmtui or nmcli. We will change the DNS resolver later.

2.5. Verify Repositories

Ensure necessary repositories are enabled.

sudo dnf repolist enabled | grep -E 'baseos|appstream'
# Should show rhel-9-for-x86_64-baseos-rpms and rhel-9-for-x86_64-appstream-rpms
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
2.6. Install and Synchronize Chrony (NTP)
sudo dnf install -y chrony
sudo systemctl enable --now chronyd

# Wait a minute or two for synchronization
sleep 60 

# Verify synchronization (look for sources marked with '*' or '+')
sudo chronyc sources
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

CRITICAL: Do not proceed if time is not synchronized. Fix NTP issues first.

2.7. Check for Conflicting Services

Ensure services like dnsmasq are not running if you plan to use FreeIPA's integrated DNS.

# sudo systemctl stop dnsmasq
# sudo systemctl disable dnsmasq
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
3. Install FreeIPA Packages
# Install the core server and the integrated DNS components
sudo dnf install -y ipa-server ipa-server-dns
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

(If using external DNS managed elsewhere, omit ipa-server-dns).

4. Configure FreeIPA Server

This is the main interactive installation step.

# Run the installation script with the flag for integrated DNS
sudo ipa-server-install --setup-dns
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

(If using external DNS, run sudo ipa-server-install without --setup-dns).

Follow the prompts carefully:

Server Host Name: Should default correctly to ipa.lab.example.com. Press Enter.

Domain Name: Should be detected as lab.example.com. Press Enter.

Realm Name: Should default to LAB.EXAMPLE.COM. Press Enter.

Directory Manager Password: Enter and confirm a strong password. SAVE THIS PASSWORD SECURELY.

IPA Admin Password: Enter and confirm a strong password for the admin user. SAVE THIS PASSWORD SECURELY.

Configure DNS Server?: Yes (default, assuming --setup-dns).

DNS forwarders: Enter IP addresses of reliable upstream DNS servers (e.g., your router, 8.8.8.8, 1.1.1.1) separated by spaces, or choose no (not recommended unless isolated network). The installer might detect from /etc/resolv.conf. Configuring forwarders is highly recommended for internet access.

Configure Reverse Zone?: Usually Yes. (Ensure a reverse zone for 192.168.1.0/24 or similar makes sense in your setup).

Review configuration: Check the displayed settings carefully.

Continue to configure the system? Type yes and press Enter.

The installation process will take several minutes (10-20+ min). It will configure many services (Kerberos, LDAP, CA, DNS, NTP, HTTPD).

Configure firewall? Answer yes (recommended).

Configure chrony? Answer yes (recommended).

Note the information displayed upon successful completion.

5. Post-Installation Steps
5.1. Configure Firewall (firewalld)

If you didn't let the installer configure the firewall, or want to verify, use these commands:

# Add required services permanently
sudo firewall-cmd --permanent \
  --add-service=http \
  --add-service=https \
  --add-service=ldap \
  --add-service=ldaps \
  --add-service=kerberos \
  --add-service=kpasswd \
  --add-service=dns \
  --add-service=ntp

# Reload firewall to apply permanent rules
sudo firewall-cmd --reload

# Verify services are active in the default zone
sudo firewall-cmd --list-all 
# (Check the 'services:' line)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
5.2. Configure DNS Resolution (Server Self-Resolution)

The FreeIPA server should now use itself for DNS.

# Set the server's primary IP address (already known)
IPA_SERVER_IP="192.168.1.202" 

# Get the active network connection name (adjust if multiple active non-loopback)
CONN_NAME=$(nmcli -t -f NAME,DEVICE c show --active | grep -v ':lo$' | head -n 1 | cut -d':' -f1)

# Configure the connection to use the IPA server IP for DNS
sudo nmcli con mod "$CONN_NAME" ipv4.dns "$IPA_SERVER_IP"
# Prevent NetworkManager from overwriting with DHCP DNS info (Important!)
sudo nmcli con mod "$CONN_NAME" ipv4.ignore-auto-dns yes

# Restart the network connection to apply changes
sudo nmcli con down "$CONN_NAME" && sudo nmcli con up "$CONN_NAME"

# Verify /etc/resolv.conf points to your IPA server IP and domain
cat /etc/resolv.conf
# Example Output:
# search lab.example.com
# nameserver 192.168.1.202
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
5.3. Authenticate as IPA Admin (Kerberos)
# Request a Kerberos ticket for the admin user
kinit admin
# Enter the IPA Admin Password you set during installation

# Verify you have a ticket
klist
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
5.4. Test DNS Resolution (from Server)
# Test forward lookup for the IPA server itself
dig @localhost $(hostname -f) A +short
# Expected Output: 192.168.1.202

# Test reverse lookup for the IPA server IP
dig @localhost -x 192.168.1.202 +short
# Expected Output: ipa.lab.example.com. (or similar PTR record)

# Test service record lookup (should return server FQDN and port)
dig @localhost _ldap._tcp.$(dnsdomainname) SRV +short 
# Expected Output: 0 100 389 ipa.lab.example.com. (or similar)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
5.5. Access the Web UI

On a client machine (like your Windows PC), edit its hosts file to map the IPA server's FQDN (ipa.lab.example.com) to its IP address (192.168.1.202) if the client doesn't use the IPA server for DNS yet (See Appendix below).

Open a web browser and navigate to https://ipa.lab.example.com.

Accept the security warning (the certificate is signed by the IPA CA, which your browser doesn't trust yet).

Log in with username admin and the IPA Admin Password.

6. Initial Usage and Next Steps

Explore the Web UI: Add users, groups, hosts.

Configure Sudo rules.

Enroll RHEL/Linux client machines using ipa-client-install.

Configure Windows clients (requires more setup).

Set up FreeIPA replicas for high availability and load balancing.

Refer to the official Red Hat Identity Management Documentation for advanced topics.

Appendix: Modifying Windows Hosts File (for Client Access)

If your regular DNS server doesn't know about your IPA server, you need to tell your Windows client how to find it manually for Web UI access:

Open Notepad as Administrator: Start -> type notepad -> Right-click -> Run as administrator.

Open Hosts File: File -> Open -> Navigate to C:\Windows\System32\drivers\etc\ -> Select "All Files (*.*)" -> Open hosts.

Add Entry: Go to the bottom and add a line:

# <IPA-Server-IP>  <IPA-Server-FQDN>
192.168.1.202    ipa.lab.example.com
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
IGNORE_WHEN_COPYING_END

Save: File -> Save.

Flush DNS Cache (Optional but recommended): Open Command Prompt (cmd) and run ipconfig /flushdns.

Disclaimer: Always test configurations in a non-production environment first. Ensure you have backups before making significant system changes.

This updated version should now perfectly match your specific environment details in all the relevant commands and examples.
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
IGNORE_WHEN_COPYING_END
