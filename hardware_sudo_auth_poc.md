# 🔐 Proof of Concept (PoC)
## End-to-End Hardware-Backed SSH + sudo Authentication on SLES 16

**Author:** Joseph Oaks  
**Audience:** Senior SysAdmin / DevOps / Security Engineering  
**Objective:** Eliminate reusable passwords for SSH and `sudo`.  
**Scope:** Use a *locally attached* YubiKey, CAC, or RSA smartcard to authenticate both SSH login and `sudo` operations on remote SLES 16 hosts.

---

## 1  Overview

### 🔎 Concept
All administrative actions require two hardware-enforced factors:

1. **Something you have:** YubiKey / CAC / Smartcard  
2. **Something you know:** PIN + Touch confirmation  

Both SSH login and `sudo` authorization are verified through **cryptographic signing by the same hardware token**, but the token never leaves the user’s workstation.

### 🔄 Mechanics
| Phase | Component | Where it Executes | Module / Tool |
|:------|:-----------|:-----------------|:--------------|
| SSH login | Smartcard key signing | Local workstation | `openssh` + PKCS #11 (`opensc`, `libykcs11`) |
| Agent forward | Forwards crypto requests | SSH tunnel | `ssh-agent` forwarding |
| `sudo` auth | PAM verifies signature | Remote SLES 16 | `pam_ssh_agent_auth.so` |

---

## 2  Pre-Requisites

### Workstation
- YubiKey, CAC, or Smartcard token with RSA or FIDO2 key  
- Working `ssh-agent` or `gpg-agent` integrated with token  
- OpenSSH 8.x or newer (supports agent forwarding)  
- Token middleware (`opensc`, `yubikey-manager`, `pcscd`)

### Remote SLES 16 Host
```bash
sudo zypper in pam_ssh_agent_auth openssh pcsc-lite
sudo systemctl enable --now sshd pcscd
```

---

## 3  Generate / Import Key Material on Workstation

### Option A — FIDO2 / YubiKey SSH Key
```bash
ssh-keygen -t ed25519-sk -C "alice@remote-sles16"
```
Touch the device when prompted.  
Public key → `~/.ssh/id_ed25519_sk.pub`

### Option B — Smartcard / CAC (PKCS #11)
```bash
ssh-keygen -D /usr/lib64/pkcs11/opensc-pkcs11.so
```
Select desired key and copy its public portion.

---

## 4  Deploy Public Key to Remote Server
```bash
ssh-copy-id -i ~/.ssh/id_ed25519_sk.pub alice@remote-sles16
```

Verify login:
```bash
ssh alice@remote-sles16
```
→ Prompt for PIN + Touch

---

## 5  Enable SSH Agent Forwarding
Local `~/.ssh/config`:
```text
Host remote-sles16
    ForwardAgent yes
    PKCS11Provider /usr/lib64/pkcs11/opensc-pkcs11.so   # For CAC/Smartcard
```

Check forwarding:
```bash
echo $SSH_AUTH_SOCK
```
Socket path → active.

---

## 6  Configure `pam_ssh_agent_auth` for sudo

### 6.1  Install and Prepare
```bash
sudo zypper in pam_ssh_agent_auth
```

### 6.2  Authorize Public Keys for sudo
```bash
sudo mkdir -p /etc/security
sudo cp /home/alice/.ssh/id_ed25519_sk.pub /etc/security/authorized_keys
sudo chmod 600 /etc/security/authorized_keys
sudo chown root:root /etc/security/authorized_keys
```

### 6.3  Edit PAM Stack (`/etc/pam.d/sudo`)
```text
#%PAM-1.0
auth    requisite pam_ssh_agent_auth.so file=/etc/security/authorized_keys
account include   common-account
session include   common-session-nonlogin
```

### 6.4  Preserve Agent Environment
Edit with `visudo`:
```bash
Defaults env_keep += "SSH_AUTH_SOCK"
```

---

## 7  Test the Full Chain

1. **Login via SSH**
   ```bash
   ssh alice@remote-sles16
   ```
   → PIN + Touch  
2. **Run sudo**
   ```bash
   sudo whoami
   ```
   → `pam_ssh_agent_auth` requests signature from forwarded agent.  
   → PIN + Touch may re-prompt based on token policy.

If token removed or agent locked → both SSH and sudo fail.

---

## 8  Security Model
| Threat | Mitigation |
|:--------|:------------|
| Stolen credentials | Private key never leaves hardware token |
| Remote compromise | `sudo` requires agent-backed signature |
| Replay attacks | Per-session SSH and PAM challenge/response |
| Insider abuse | Every `sudo` event is hardware-bound and auditable |

---

## 9  Auditing and Logging
Add audit rule:
```bash
echo "-w /etc/pam.d/sudo -p wa -k agent-auth" | sudo tee /etc/audit/rules.d/agent-auth.rules
sudo augenrules --load
```

Query:
```bash
sudo ausearch -k agent-auth
```

---

## 10  Recovery Strategy
- Maintain a secondary admin token and key in `/etc/security/authorized_keys`.  
- Restrict root SSH logins to console or break-glass accounts.  
- Back up authorized keys securely (never private keys).

---

## 11  Optional Enhancements
| Feature | Method |
|:---------|:--------|
| CA-signed cert (CAC) | Use `TrustedUserCAKeys` and cert-based SSH login |
| Multi-person approval for sudo | Combine `pam_ssh_agent_auth` with TPI policy wrapper |
| Central revocation | Use `RevokedKeys` or HSM CA |
| FIPS 140-2 compliance | FIPS-validated PKCS #11 provider (DoD CAC middleware) |

---

## 12  Verification Checklist
| Check | Command | Expected Result |
|:------|:---------|:----------------|
| Agent visible | `echo $SSH_AUTH_SOCK` | Path exists |
| Key listed | `ssh-add -l` | Smartcard key shown |
| sudo works | `sudo whoami` | Auth via token |
| Audit trail | `ausearch -k agent-auth` | Entries present |

---

## 13  Summary
| Goal | Achieved |
|------|-----------|
| Remote SSH auth via hardware token | ✅ |
| `sudo` auth tied to same token | ✅ |
| Token never on remote host | ✅ |
| No password fallback | ✅ |
| Auditable chain of trust | ✅ |

---

## 14  References
- **OpenSSH Agent Forwarding:** <https://man.openbsd.org/ssh-agent>  
- **pam_ssh_agent_auth:** <https://github.com/jbeverly/pam_ssh_agent_auth>  
- **OpenSC Project:** <https://github.com/OpenSC/OpenSC>  
- **SUSE PAM Guide:** <https://documentation.suse.com/sles/16/html/SLES-all/cha-pam.html>  

---

> **Result:**  
> Remote SLES 16 systems now enforce **hardware-backed sudo authorization** using your local YubiKey / CAC / RSA smartcard, guaranteeing cryptographic non-repudiation and eliminating passwords from the administrative path.

