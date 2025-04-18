# Joining a RHEL Host to FreeIPA Using RHEL System Role (`ipa_client`)

This guide explains how to join a RHEL 9 host to a FreeIPA domain using the officially supported Red Hat System Role: `ipa_client`.

---

## 🔧 Prerequisites

- A functional FreeIPA server (e.g., `ipa.lab.example.com`).
- IPA admin credentials.
- RHEL 8/9 client(s) enrolled with Red Hat Subscription.
- Ansible control node installed and configured.
- Optional: Use `ansible-vault` to securely store the IPA admin password.

---

## 🛠️ Step-by-Step Instructions

### 1. ✅ Install RHEL System Roles (if not already installed)

```bash
sudo dnf install -y rhel-system-roles
```

Or install via Ansible Galaxy:

```bash
ansible-galaxy collection install redhat.rhel_system_roles
```

---

### 2. ✅ Create Your Ansible Inventory

**`inventory.ini`**
```ini
[freeipa_clients]
client1.lab.example.com
client2.lab.example.com
```

---

### 3. ✅ Create a Secure Vault File (Recommended)

```bash
ansible-vault create group_vars/all/vault.yml
```

**Contents:**
```yaml
vault_ipa_password: SuperSecureAdminPassword
```

Use `--ask-vault-pass` or a vault password file when running the playbook.

---

### 4. ✅ Create the Ansible Playbook

**`ipa_client_join.yml`**
```yaml
---
- name: Join host to FreeIPA domain using RHEL system role
  hosts: freeipa_clients
  become: true
  vars:
    ipa_client_domain: lab.example.com
    ipa_client_realm: LAB.EXAMPLE.COM
    ipa_client_servers:
      - ipa.lab.example.com
    ipa_client_force: true
    ipa_client_mkhomedir: true
    ipa_client_enable_dns_updates: true
    ipa_client_configure_ntp: true
    ipa_client_ntp_servers:
      - ipa.lab.example.com
    ipa_client_password: "{{ vault_ipa_password }}"
    ipa_client_principal: admin

  roles:
    - redhat.rhel_system_roles.ipa_client
```

---

### 5. ✅ Run the Playbook

```bash
ansible-playbook -i inventory.ini ipa_client_join.yml --ask-vault-pass
```

---

## 🧪 Verification After Join

On each client:
```bash
kinit admin
klist
ipa user-find admin
```

You should see a valid Kerberos ticket and FreeIPA user data without errors.

---

## 🔄 How the Role Maps to `ipa-client-install` Prompts

When using the system role, it bypasses the following interactive prompts and sets values automatically based on the variables you define:

| `ipa-client-install` Prompt                                | Handled by Role Variable                    | Example Value                 |
|------------------------------------------------------------|---------------------------------------------|-------------------------------|
| Proceed with fixed values and no DNS discovery? [no]       | `ipa_client_servers`                        | `ipa.lab.example.com`         |
| Do you want to configure chrony? [no]                      | `ipa_client_configure_ntp`                  | `true`                        |
| Enter NTP source server address                            | `ipa_client_ntp_servers`                    | `ipa.lab.example.com`         |
| Client hostname / DNS domain / Realm / BaseDN             | `ipa_client_domain`, `ipa_client_realm`     | `lab.example.com`, etc.       |
| IPA Server                                                 | `ipa_client_servers`                        | `ipa.lab.example.com`         |
| Continue to configure the system with these values? [no]   | Role uses `--unattended`                    | ✅ Automatically accepted     |

---

## 🔐 Optional Enhancements
- Bundle with a firewall configuration role.
- Integrate user creation or UID setting with `ipa` command module.
- Extend to support SSH key injection or sudo policy automation.

* Done!

---

## 📚 References
- [Red Hat System Roles Docs](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_identity_management/index)
- `man ipa-client-install`
- `man chrony.conf`

---


## To add a user via CLI:
```
ipa user-add john --first=John --last=Smith --password

# When prompted, enter the password twice

```

## To modify a user to never expire:
```
ipa user-mod testuser --password-expiration=
```

## To modify a user's UID:
```
ipa user-mod john --uid=10501
```
