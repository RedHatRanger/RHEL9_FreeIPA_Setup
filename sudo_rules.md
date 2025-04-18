# 🧑‍💻 Granting Sudo Access to FreeIPA Users

This guide covers how to assign sudo privileges to a FreeIPA user using both the **Web UI** and **CLI**.

---

## ✅ Requirements
- An existing FreeIPA environment with server and enrolled clients
- Admin access to the FreeIPA Web UI or CLI
- A target user (e.g., `john`) created in IPA
- SSSD running on the IPA-enrolled client(s)

---

## 🌐 Using the FreeIPA Web UI

### 1. Log In to Web UI
Visit: `https://ipa.lab.example.com`  
Login as `admin` or another authorized admin user

### 2. Navigate to Sudo Rules
- Go to the **Policy** tab (top navigation)
- Click **Sudo Rules** from the sidebar

### 3. Create a New Sudo Rule
- Click **Add**
- Rule Name: `allow_sudo_john`
- Description: `Full sudo access for john`
- Click **Add and Edit**

### 4. Configure the Rule

#### Under `Who>Users`:
- Click **Add**, then select `john`

#### Under `Access this host>Hosts`:
- Click **Add**, then select the enrolled host (e.g., `controller.lab.example.com` or select `Any host`)

#### Under `Run Commands>Allow>Sudo Allow Commands`:
- Click **Add**, then choose **Any Command** (e.g., `/usr/bin/locate`)

### 5. Save the Rule
Click **Save** at the top to activate the rule.

### 6. Refresh Policy on the Client:
On the client machine (e.g., `controller.lab.example.com`):
```bash
sudo systemctl restart sssd
```

### 7. Test Sudo Access
Log in as the user:
```bash
su - john
```
```
sudo locate chrony.conf        # As a test
```

Expected output:
```
/etc/chrony.conf
/usr/lib/sysusers.d/chrony.conf
/usr/share/man/man5/chrony.conf.5.gz
/var/lib/awx/.local/share/containers/storage/overlay/a91b7d030abc1cabfda5fe7d63f5899235001ef54041eb94790e42e178210c32/diff/usr/share/ansible/collections/ansible_collections/redhat/rhel_system_roles/roles/timesync/templates/chrony.conf.j2
```

---

## 🛡️ Notes
- Apply sudo access via **groups** instead of individual users for scalability
- Consider setting command restrictions for security (instead of allowing all)
- Audit sudo logs using `journalctl` or `/var/log/secure`

---

## 📚 References
- [FreeIPA Sudo Rules](https://www.freeipa.org/page/V4/SUDO_Integration)
- `man sssd-sudo`
- `man ipa sudorule-add`
