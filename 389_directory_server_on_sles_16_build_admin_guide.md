# 389 Directory Server on SLES 16 — Build & Administration Guide

> **Audience:** Senior Linux System Administrators / DevOps Engineers  
> **Platform:** SUSE Linux Enterprise Server (SLES) 16  
> **Directory Service:** 389 Directory Server (389-ds)  
> **Management Interface:** Cockpit 389 Directory Server Plugin (native)

---

## 1. Overview

This guide documents a **clean, repeatable, production‑grade deployment** of a standalone **389 Directory Server** instance on **SLES 16**.

The final architecture **intentionally uses Cockpit’s 389 Directory Server plugin** as the sole graphical management interface. Earlier experimentation with phpLDAPadmin was **fully retired** and is **not part of the supported design**.

---

## 2. Design Decisions (Intentional)

- **Cockpit 389 DS plugin is preferred**
- Management tightly integrated with systemd, SELinux, and 389‑ds tooling
- CLI remains the authoritative control surface

This aligns with **SUSE‑supported** and **long‑term maintainable** practices.

---

## 3. Prerequisites

- SLES 16 system (registered to SCC or RMT)
- Root or sudo access
- Static hostname/IP recommended
- Firewall access for LDAP (389 / 636)

Verify environment:

```bash
cat /etc/os-release
getenforce
```

---

## 4. Repository Enablement

### 4.1 Register System

```bash
SUSEConnect -r <SLES_REGISTRATION_CODE>
```

If re‑registering a system:

```bash
SUSEConnect -d
SUSEConnect --cleanup
```

### 4.2 Enable PackageHub

```bash
SUSEConnect -p PackageHub/16.0/x86_64
zypper refresh
```

---

## 5. Package Installation

```bash
zypper install -y \
  cockpit \
  389-ds-base \
  selinux-policy \
  selinux-tools \
  auditd
```

Optional diagnostics:

```bash
zypper install -y pamtester
```

Verify:

```bash
rpm -q 389-ds-base selinux-policy selinux-tools
```

---

## 6. SELinux Configuration

For initial deployment, **permissive** is recommended.

```bash
vi /etc/selinux/config
```

```text
SELINUX=permissive
SELINUXTYPE=targeted
```

Apply immediately if required:

```bash
setenforce 0
```

---

## 7. Create 389 Directory Server Instance

### 7.1 Instance Definition File

```bash
cat > /root/<INSTANCE_NAME>.inf <<'EOF'
[general]
config_version = 2

[slapd]
instance_name = <INSTANCE_NAME>
root_password = <PASSWORD>
suffix = dc=example,dc=com
secure_port = 636
self_sign_cert = True
EOF
```

### 7.2 Create Instance

```bash
dscreate from-file /root/<INSTANCE_NAME>.inf
```

### 7.3 Enable and Start Service

```bash
systemctl enable --now dirsrv@<INSTANCE_NAME>
systemctl status dirsrv@<INSTANCE_NAME>
```

Verify:

```bash
dsctl -l
```

---

## 8. Initialize Directory Structure

### 8.1 Base Suffix

`base.ldif`

```ldif
dn: dc=example,dc=com
objectClass: top
objectClass: domain
dc: example
description: Base domain entry
```

```bash
ldapadd -H ldap://127.0.0.1 -x \
  -D "cn=Directory Manager" -W \
  -f /root/base.ldif
```

### 8.2 Organizational Units

`ou_structure.ldif`

```ldif
dn: ou=People,dc=example,dc=com
objectClass: organizationalUnit
ou: People


dn: ou=Groups,dc=example,dc=com
objectClass: organizationalUnit
ou: Groups
```

```bash
ldapadd -H ldap://127.0.0.1 -x \
  -D "cn=Directory Manager" -W \
  -f /root/ou_structure.ldif
```

---

## 9. Create LDAP Admin User

### 9.1 Generate Password Hash

```bash
slappasswd -s <PASSWORD>
```

### 9.2 Admin User LDIF

```ldif
dn: uid=admin,ou=People,dc=example,dc=com
objectClass: inetOrgPerson
uid: admin
cn: Admin User
sn: User
mail: admin@example.com
userPassword: <SSHA_HASH>
```

```bash
ldapadd -H ldap://127.0.0.1 -x \
  -D "cn=Directory Manager" -W \
  -f /root/admin_user.ldif
```

---

## 10. Access Control (ACI)

### 10.1 Enable ACI Engine

```bash
dsconf <INSTANCE_NAME> config replace nsslapd-accesscontrol=on
systemctl restart dirsrv@<INSTANCE_NAME>
```

### 10.2 Admin Full‑Access ACI

```ldif
dn: dc=example,dc=com
changetype: modify
add: aci
aci: (target="ldap:///dc=example,dc=com")
 (targetattr="*")
 (version 3.0; acl "Admin full access";
 allow (all)
 userdn="ldap:///uid=admin,ou=People,dc=example,dc=com";)
```

```bash
ldapmodify -H ldap://127.0.0.1 -x \
  -D "cn=Directory Manager" -W \
  -f /root/admin_full_access.ldif
systemctl restart dirsrv@<INSTANCE_NAME>
```

Validate:

```bash
ldapwhoami -H ldap://127.0.0.1 -x \
  -D "uid=admin,ou=People,dc=example,dc=com" -w <PASSWORD>
```

---

## 11. Firewall Configuration

```bash
firewall-cmd --add-port=389/tcp --permanent
firewall-cmd --add-port=636/tcp --permanent
firewall-cmd --reload
```

---

## 12. Cockpit 389 Directory Server Plugin

> **Note:** On SLES 16 there is **no packaged `cockpit-389-ds` RPM**. The Cockpit 389 Directory Server UI is deployed **manually** from the upstream source tree.

This section documents the **supported, repeatable manual install** used for this system.

---

### 12.1 Enable Cockpit and add 389-DS Plugin

Enable Cockpit:

```bash
systemctl enable --now cockpit.socket
```

Verify:

```bash
cockpit --version
```

---

### 12.2 Retrieve 389-ds Source (Cockpit UI)

```bash
cd /usr/src
wget https://fossies.org/linux/misc/389-ds-base-389-ds-base-3.1.4.tar.gz
tar xfz 389-ds-base-389-ds-base-3.1.4.tar.gz
cd /usr/src/389-ds-base-389-ds-base-3.1.4/src/cockpit/389-console
npm install
npm run build
mkdir -p /usr/share/cockpit/389-console
cp -r  dist/ pkg/ src/ org.port389.cockpit_console.metainfo.xml /usr/share/cockpit/389-console/
cd /usr/share/cockpit/389-console
cp dist/manifest.json .
cp dist/index.css .
cp dist/index.html .
cp dist/index.js .
```

Navigate to the Cockpit console source:

```bash
cd 389-ds-base-389-ds-base-3.1.4/src/cockpit/389-console
```

---

### 12.3 Build Cockpit 389 Console

```bash
npm install
npm run build
```

---

### 12.4 Install Console into Cockpit

After the build completes, additional artifacts must be staged and the **Cockpit manifest adjusted**.

#### 12.4.1 Install Console Files

```bash
mkdir -p /usr/share/cockpit/389-console

cp -r dist/ pkg/ src/ org.port389.cockpit_console.metainfo.xml \
  /usr/share/cockpit/389-console/

# Explicitly stage top-level assets
cp dist/manifest.json /usr/share/cockpit/389-console/
cp dist/index.css     /usr/share/cockpit/389-console/
cp dist/index.html    /usr/share/cockpit/389-console/
cp dist/index.js      /usr/share/cockpit/389-console/
```

#### 12.4.2 Update Cockpit Manifest Requirement

Edit the manifest to ensure compatibility with the Cockpit version shipped in SLES 16.

```bash
vi /usr/share/cockpit/389-console/manifest.json
```

Modify the `cockpit` requirement:

```json
{
  "cockpit": ">=137"
}
```

> This change is required for the plugin to be recognized by the Cockpit bridge.

#### 12.4.3 Permissions and Restart

```bash
chmod -R 755 /usr/share/cockpit/389-console
systemctl restart cockpit
```

Verify registration:

```bash
cockpit-bridge --packages | grep 389
```

---

### 12.5 Access Cockpit 389 Console

```text
https://<host>:9090
```

Navigate to:

```text
Cockpit → 389 Directory Server
```

Capabilities:

- Instance lifecycle management
- Backend and suffix inspection
- Log review
- TLS status and configuration visibility
- Safe administrative operations

---

## 13. Validation Checklist

- [ ] `systemctl status dirsrv@<INSTANCE_NAME>`
- [ ] `ldapsearch -x -H ldap://localhost -b dc=example,dc=com`
- [ ] Admin user bind succeeds
- [ ] ACI enforcement verified
- [ ] Cockpit 389 DS plugin accessible

---

## 14. Operational Notes / Lessons Learned

- **ACI is case‑sensitive** (`ou=People` vs `ou=people`)
- `nsslapd-accesscontrol` must be enabled explicitly
- Prefer SSHA password storage
- Cockpit replaces generic LDAP web UIs cleanly
- SELinux permissive simplifies initial bring‑up
