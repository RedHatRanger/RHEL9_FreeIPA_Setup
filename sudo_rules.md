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

#### Users tab
- Click **Add**, then select `john`

#### Commands tab
- Click **Add**, then choose **Any Command** (or set Command Category to `all`)

#### Hosts tab
- Click **Add**, then select the enrolled host (e.g., `controller.lab.example.com`)

#### Options tab
- (Optional) Set Sudo order: `1`

### 5. Save the Rule
Click **Save** to activate the rule.

### 6. Refresh Policy on the Client
On the client machine (e.g., `controller.lab.example.com`):
```bash
sudo systemctl restart sssd
```

### 7. Test Sudo Access
Log in as the user:
```bash
su - john
sudo whoami
```
Expected output:
```
root
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
