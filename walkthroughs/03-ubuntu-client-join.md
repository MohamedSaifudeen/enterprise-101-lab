# 03 – Ubuntu Desktop AD Join (Samba + Winbind)

**Objective**  
Join Ubuntu 22.04 (`project-x-linux-client`) to the AD domain `corp.project-x-dc.com` (10.0.0.5) and enable domain‑user logins.

---

## 1. Prerequisites
- Ubuntu 22.04 VM, static IP 10.0.0.101/24, gateway 10.0.0.1, DNS 10.0.0.5  
- AD domain reachable at `corp.project-x-dc.com`  
- AD “Administrator” credentials  

---

## 2. Install & Configure

```bash
# 2.1 Install required packages
sudo apt update
sudo apt install -y samba winbind libnss-winbind libpam-winbind krb5-config krb5-user

# 2.2 Hostname & hosts file
sudo hostnamectl set-hostname linux-client
echo "10.0.0.5 corp.project-x-dc.com" | sudo tee -a /etc/hosts

# 2.3 Fix DNS
sudo rm /etc/resolv.conf
echo "nameserver 10.0.0.5" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf

# 2.4 Samba config
sudo mv /etc/samba/smb.conf{,.org}
cat << 'EOF' | sudo tee /etc/samba/smb.conf
[global]
  workgroup = CORP
  realm = CORP.PROJECT-X-DC.COM
  security = ads
  kerberos method = secrets and keytab
  template shell = /bin/bash
  winbind use default domain = no
  winbind enum users = yes
  winbind enum groups = yes
  winbind separator = +
  idmap config * : backend = autorid
  idmap config * : range = 1000000-19999999
EOF

# 2.5 NSS
sudo sed -i 's/^\(passwd:\s*compat\).*/\1 winbind/' /etc/nsswitch.conf
sudo sed -i 's/^\(group:\s*compat\).*/\1 winbind/'  /etc/nsswitch.conf

---

## 3. PAM & Domain Join

# 3.1 Enable domain auth + home-dir creation
sudo pam-auth-update --enable mkhomedir

# 3.2 Restart Winbind, join domain
sudo systemctl restart winbind
sudo net ads join -U Administrator      # enter @Deeboodah1!
```

---
```bash
## 4. Verify & Troubleshoot


wbinfo -u               # should list CORP+janed
getent passwd CORP+janed  # should show a passwd entry
su - CORP+janed           # should drop you into her shell
```
```
**Common issues & fixes**

* **No DNS / Kerberos lookup** → ensure `/etc/resolv.conf` points at the DC and is immutable.
* **`getent` returns nothing** → manually map home & shell:

  ```bash
  sudo mkdir -p /home/CORP/janed
  sudo useradd -M -d /home/CORP/janed -s /bin/bash CORP+janed

* **Account locked in AD** → unlock in AD Users & Computers; reset password.

---
## 5. Snapshot & Next

1. Take a VirtualBox snapshot: **Ubuntu‑AD‑Integrated**
2. Grant sudo if desired:

   ```bash
   sudo usermod -aG sudo CORP+janed
   ```
```
```
