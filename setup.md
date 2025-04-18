FreeIPA Server & Client Installation Guide (RHEL 9.5)This guide provides step-by-step instructions for installing a FreeIPA server with integrated DNS on Red Hat Enterprise Linux (RHEL) 9.5, followed by instructions for enrolling RHEL 9.5 clients.Environment Details:RHEL Version: 9.5IPA Server FQDN: ipa.lab.example.comIPA Server IP: 192.168.1.202IPA Domain: lab.example.comIPA Realm: LAB.EXAMPLE.COMPart 1: FreeIPA Server Installation (ipa.lab.example.com)1.1. Important ConsiderationsFQDN: The server must have ipa.lab.example.com as its hostname.Static IP: The server requires the static IP 192.168.1.202.Integrated DNS: This guide assumes using FreeIPA's integrated BIND DNS (--setup-dns). If using external DNS, omit --setup-dns and the ipa-server-dns package, and manually configure required DNS records beforehand.Resources: Minimum 4 GB RAM recommended.Time (NTP): Accurate time synchronization via chronyd is critical before installation.Firewall: firewalld is required. The installer can configure it automatically (recommended).SELinux: Must be in Enforcing mode.Subscription: The system must be registered and subscribed.Passwords: Prepare strong passwords for the Directory Manager (cn=Directory Manager) and the IPA admin user. Store them securely.1.2. Server PrerequisitesPerform these steps on the target RHEL 9.5 server (ipa.lab.example.com).1.2.1. Update Systemsudo dnf update -y
# Reboot if kernel or core components were updated
sudo reboot
1.2.2. Set Static Hostname (FQDN)sudo hostnamectl set-hostname ipa.lab.example.com

# Verify
hostnamectl status
hostname -f # Output must be ipa.lab.example.com
1.2.3. Configure /etc/hostsEnsure local resolution works before DNS is fully set up.# Edit the hosts file (e.g., with vi or nano)
sudo vi /etc/hosts

# Ensure this line exists, placed after 127.0.0.1 and ::1 entries:
# <Static-IP>      <FQDN>                 <Short-Hostname>
192.168.1.202    ipa.lab.example.com    ipa

# Save and close the file.

# Verify resolution
getent hosts $(hostname -f)
# Expected output: 192.168.1.202    ipa.lab.example.com
1.2.4. Verify Network ConfigurationConfirm the static IP (192.168.1.202), netmask, gateway, and temporary external DNS servers (e.g., 8.8.8.8) are correctly configured using nmtui or nmcli. The DNS settings will be changed later.1.2.5. Verify Repositoriessudo dnf repolist enabled | grep -E 'baseos|appstream'
# Should show rhel-9-for-x86_64-baseos-rpms and rhel-9-for-x86_64-appstream-rpms
1.2.6. Install and Synchronize Chrony (NTP)sudo dnf install -y chrony
sudo systemctl enable --now chronyd

# Wait ~1 minute for synchronization
sleep 60

# Verify synchronization (look for '*' or '+' prefixes)
sudo chronyc sources
CRITICAL: Do not proceed if time is not synchronized. Resolve NTP issues first.1.2.7. Check for Conflicting ServicesEnsure services like dnsmasq are stopped and disabled if using FreeIPA's integrated DNS.# Example for dnsmasq:
# sudo systemctl stop dnsmasq
# sudo systemctl disable dnsmasq
1.3. Install FreeIPA Server Packages# Install core server and integrated DNS components
sudo dnf install -y ipa-server ipa-server-dns
(Omit ipa-server-dns if using external DNS)1.4. Run the FreeIPA Server Installation ScriptThis is the main interactive setup.# Run with the integrated DNS setup flag
sudo ipa-server-install --setup-dns
(Omit --setup-dns if using external DNS)Follow the prompts carefully:Server Host Name: Verify ipa.lab.example.com. Press Enter.Domain Name: Verify lab.example.com. Press Enter.Realm Name: Verify LAB.EXAMPLE.COM. Press Enter.Directory Manager Password: Enter and confirm a strong password. Save securely.IPA Admin Password: Enter and confirm a strong password for the admin user. Save securely.Configure DNS Server?: yes (default with --setup-dns).DNS forwarders: Enter upstream DNS server IPs (e.g., 8.8.8.8 1.1.1.1) or choose no if appropriate. Using forwarders is recommended.Configure Reverse Zone?: Usually yes. Ensure a reverse zone (e.g., 1.168.192.in-addr.arpa) is appropriate.Review configuration: Check settings carefully.Continue to configure the system?: Type yes and press Enter.The installation will take several minutes.Configure firewall?: Answer yes (recommended).Configure chrony?: Answer yes (recommended).Note the information displayed upon successful completion.1.5. Server Post-Installation Steps1.5.1. Verify Firewall (firewalld)If you didn't let the installer configure the firewall, or want to verify:# List active services in the default zone
sudo firewall-cmd --list-services

# Expected services include: http, https, ldap, ldaps, kerberos, kpasswd, dns, ntp

# If needed, add them permanently and reload:
# sudo firewall-cmd --permanent --add-service={http,https,ldap,ldaps,kerberos,kpasswd,dns,ntp}
# sudo firewall-cmd --reload
1.5.2. Configure Server DNS Resolution (Self-Resolution)The IPA server should use itself for DNS.# Define server IP
IPA_SERVER_IP="192.168.1.202"

# Get the primary active network connection name (adjust if needed)
CONN_NAME=$(nmcli -g NAME,DEVICE c show --active | grep -v ':lo$' | head -n 1 | cut -d':' -f1)

# Configure the connection to use only the IPA server for DNS
sudo nmcli con mod "$CONN_NAME" ipv4.dns "$IPA_SERVER_IP"
# Prevent NetworkManager from using DHCP-provided DNS
sudo nmcli con mod "$CONN_NAME" ipv4.ignore-auto-dns yes

# Restart the network connection to apply changes
sudo nmcli con down "$CONN_NAME" && sudo nmcli con up "$CONN_NAME"

# Verify /etc/resolv.conf points ONLY to your IPA server
cat /etc/resolv.conf
# Expected Output:
# search lab.example.com
# nameserver 192.168.1.202
1.5.3. Authenticate as IPA Admin (Kerberos)# Request a Kerberos ticket for the admin user
kinit admin
# Enter the IPA Admin Password set during installation

# Verify the ticket
klist
1.5.4. Test DNS Resolution (from Server)# Test forward lookup
dig @localhost $(hostname -f) A +short
# Expected: 192.168.1.202

# Test reverse lookup
dig @localhost -x 192.168.1.202 +short
# Expected: ipa.lab.example.com.

# Test SRV record lookup
dig @localhost _ldap._tcp.$(dnsdomainname) SRV +short
# Expected: 0 100 389 ipa.lab.example.com. (or similar priority/weight)
1.5.5. Access the Web UIClient DNS/Hosts: Ensure the machine you're browsing from can resolve ipa.lab.example.com to 192.168.1.202. Either configure it to use the IPA server for DNS or add a temporary entry to its local hosts file (see Appendix).Navigate: Open a web browser to https://ipa.lab.example.com.Accept Certificate: Accept the security warning (the certificate is signed by the IPA CA, not yet trusted by your browser).Login: Use username admin and the IPA Admin Password.Part 2: FreeIPA Client Installation (RHEL 9.5)Perform these steps on each RHEL 9.5 client machine you want to enroll in the lab.example.com domain.2.1. Client Prerequisites2.1.1. Update Systemsudo dnf update -y
sudo reboot # If needed
2.1.2. Network Configuration & DNSThe client must be able to resolve the IPA server's FQDN (ipa.lab.example.com) and the domain's SRV records.Recommended: Configure the client to use the IPA server (192.168.1.202) as its primary DNS server. You can use nmtui or nmcli similar to step 1.5.2, but point DNS to the IPA server IP.# Example using nmcli (replace CONN_NAME if needed):
IPA_SERVER_IP="192.168.1.202"
CONN_NAME=$(nmcli -g NAME,DEVICE c show --active | grep -v ':lo$' | head -n 1 | cut -d':' -f1)
sudo nmcli con mod "$CONN_NAME" ipv4.dns "$IPA_SERVER_IP"
sudo nmcli con mod "$CONN_NAME" ipv4.ignore-auto-dns yes # If using static IP
sudo nmcli con down "$CONN_NAME" && sudo nmcli con up "$CONN_NAME"
# Verify /etc/resolv.conf points to 192.168.1.202
cat /etc/resolv.conf
Verify DNS resolution works before proceeding:dig ipa.lab.example.com A +short # Should return 192.168.1.202
dig _ldap._tcp.lab.example.com SRV +short # Should return IPA server SRV record
2.1.3. Time Synchronization (NTP)Ensure chronyd is installed, enabled, running, and synchronized (ideally with the IPA server itself, which ipa-client-install often configures).sudo dnf install -y chrony
sudo systemctl enable --now chronyd
sleep 10 # Allow time to start
sudo chronyc sources
2.1.4. HostnameWhile not strictly required for joining, it's good practice for clients to have unique hostnames. Use sudo hostnamectl set-hostname <client-name>.lab.example.com.2.2. Install IPA Client Packagessudo dnf install -y ipa-client
2.3. Run the Client Installation Scriptsudo ipa-client-install --mkhomedir --enable-dns-updates --server=ipa.lab.example.com --domain=lab.example.com --realm=LAB.EXAMPLE.COM --force-join
--mkhomedir: Creates home directories for IPA users on first login.--enable-dns-updates: Allows the client to securely update its DNS record on the IPA server (requires integrated DNS).--server, --domain, --realm: Explicitly specifies connection details (reduces reliance on DNS discovery).--force-join: Useful if re-enrolling a host. Can be omitted on first attempt.You will be prompted for the admin user's password (or another user with enrollment privileges) to authorize the client enrollment.2.4. Client Post-Installation Steps2.4.1. Verify Enrollment# Check IPA status
ipa user-find admin # Should succeed without asking for password if ticket cache is valid

# Try logging in as an IPA user
# 1. Create a test user in the IPA Web UI (e.g., 'testuser')
# 2. On the client: su - testuser
#    Enter the test user's password.
#    Check home directory: pwd (should be /home/testuser)
#    Exit back to root/sudo user: exit
2.4.2. Client FirewallThe ipa-client-install script generally handles necessary client-side configurations (like NTP). Clients typically initiate connections to the server and don't require incoming ports opened unless running specific services managed by IPA. Ensure your client's firewall allows outbound connections to the required ports on the server (192.168.1.202).Appendix: Modifying Windows Hosts FileUse this temporary workaround if a Windows machine needs to access the IPA Web UI (https://ipa.lab.example.com) but is not yet configured to use the IPA server for DNS. The proper long-term solution is correct DNS configuration.Open Notepad as Administrator: Start -> type notepad -> Right-click -> Run as administrator.Open Hosts File: File -> Open -> Navigate to C:\Windows\System32\drivers\etc\ -> Select "All Files (.)" -> Open hosts.Add Entry: Go to the bottom and add the line:192.168.1.202    ipa.lab.example.com
Save: File -> Save.Flush DNS Cache (Recommended): Open Command Prompt (cmd) and run ipconfig /flushdns.Disclaimer: Always test configurations in a non-production environment first. Ensure you have backups before making significant system changes. Refer to the official Red Hat Identity Management documentation for more advanced topics.
