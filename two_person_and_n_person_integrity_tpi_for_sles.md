# Two-Person and N-Person Integrity (TPI) for SLES

> **Audience:** Senior Linux System Administrators / DevOps Engineers  
> **Platform:** SUSE Linux Enterprise Server (SLES) 15/16  
> **Scope:** Local enforcement of separation-of-duties for critical commands

---

## 1. Overview

This document describes a **lightweight, host-local implementation of Two-Person Integrity (TPI)** and its extension to **N-Person (quorum-based) Integrity** for critical administrative operations on SLES systems.

The approach is intentionally:
- Simple
- Auditable
- Dependency-free
- Suitable for on-prem, air-gapped, or regulated environments

It does **not** replace enterprise Privileged Access Management (PAM) platforms, but provides a strong control where such tooling is unavailable or inappropriate.

---

## 2. Use Cases

Typical use cases include:

- Restarting directory services (LDAP, IdM, authentication)
- Modifying security controls (SELinux, firewall, audit)
- High-impact system changes
- Enforcing separation-of-duties on single hosts

---

## 3. Design Principles

- **No single admin can act alone**
- **Human approval is explicit** (no background daemons)
- **Approvals are time-bound**
- **Identities are enforced** (no double-approval)
- **Commands must match exactly**

---

## 4. Two-Person Integrity (Baseline)

### 4.1 Script Location

```
/usr/local/sbin/tpi_exec
```

This location ensures the wrapper is:
- Outside vendor-managed directories
- Accessible to privileged users
- Easy to audit

---

### 4.2 Two-Person TPI Script

```bash
#!/bin/bash
# /usr/local/sbin/tpi_exec - Enforce Two-Person Integrity for critical operations

CMD="$@"
LOCKFILE="/var/lock/tpi_action.lock"
TIMEOUT=300  # seconds

timestamp() { date '+%Y-%m-%d %H:%M:%S'; }

if [ -f "$LOCKFILE" ]; then
    echo "[TPI] $(timestamp): Second authorization detected. Executing: $CMD"
    rm -f "$LOCKFILE"
    exec $CMD
else
    echo "[TPI] $(timestamp): First approver recorded for '$CMD'"
    echo "Run again within $TIMEOUT seconds (by another authorized user) to execute."
    echo "Lock file: $LOCKFILE"
    echo "$USER $(timestamp)" > "$LOCKFILE"
    chmod 600 "$LOCKFILE"
    (
      sleep $TIMEOUT
      if [ -f "$LOCKFILE" ]; then
          rm -f "$LOCKFILE"
          echo "[TPI] Timeout expired for $CMD"
      fi
    ) & disown
    exit 0
fi
```

---

### 4.3 Operational Flow (Two-Person)

1. **First authorized user** runs:
   ```bash
   tpi_exec systemctl restart <SERVICE>
   ```
2. Request is recorded; command does not execute
3. **Second authorized user** runs the *same command* within the timeout window
4. Command executes immediately

If the timeout expires, approval is discarded.

---

### 4.4 Security Properties

- Lockfile owned by root, mode `600`
- Requires sudo/root to execute protected commands
- No stored secrets
- Clear temporal boundary for approval

---

## 5. Limitations of Two-Person Model

- Cannot enforce more than two approvers
- Cannot prevent the same user from approving twice
- No quorum or role differentiation

These limitations are addressed by the N-Person model.

---

## 6. N-Person (Quorum-Based) Integrity

### 6.1 Concept

N-Person Integrity requires **N distinct user approvals** before a command is executed.

Example:
- 3-of-3 for production changes
- 2-of-3 for emergency operations

---

### 6.2 Design Requirements

- Track unique approvers
- Reject duplicate approvals
- Enforce command consistency
- Require threshold before execution
- Clean up on timeout

---

### 6.3 Example: Three-Person Integrity Script

```bash
#!/bin/bash
# /usr/local/sbin/tpi_exec - Three-Person Integrity Example

CMD="$@"
LOCKFILE="/var/lock/tpi_action.lock"
TIMEOUT=300
REQUIRED=3

mkdir -p /var/lock

timestamp() { date '+%Y-%m-%d %H:%M:%S'; }

# Initialize lockfile if missing
[ -f "$LOCKFILE" ] || echo "CMD:$CMD" > "$LOCKFILE"

# Reject mismatched commands
if ! grep -q "^CMD:$CMD$" "$LOCKFILE"; then
    echo "[TPI] Different command already pending approval"
    exit 1
fi

# Prevent duplicate approval
if grep -q "^$USER$" "$LOCKFILE"; then
    echo "[TPI] User $USER has already approved"
    exit 1
fi

# Record approval
echo "$USER" >> "$LOCKFILE"
COUNT=$(grep -vc '^CMD:' "$LOCKFILE")

echo "[TPI] $(timestamp): Approval $COUNT/$REQUIRED recorded"

# Execute if threshold met
if [ "$COUNT" -ge "$REQUIRED" ]; then
    echo "[TPI] Approval threshold met. Executing: $CMD"
    rm -f "$LOCKFILE"
    exec $CMD
fi

# Timeout cleanup
(
  sleep $TIMEOUT
  rm -f "$LOCKFILE"
  echo "[TPI] Timeout expired for $CMD"
) & disown

exit 0
```

---

## 7. Operational Guidance

### 7.1 When to Use TPI

Use TPI when:
- You need **local separation-of-duties**
- Enterprise PAM is unavailable
- Changes must require human quorum

### 7.2 When Not to Use TPI

Do **not** rely on this for:
- Large fleets
- Compliance requiring centralized approvals
- Hardware-backed quorum enforcement

---

## 8. Hardening Recommendations

- Restrict `tpi_exec` via sudoers
- Log executions to journald
- Make lock directory root-owned
- Pair with immutable audit logs

---

## 9. Sudoers Integration (Command-Level Enforcement)

To ensure that critical operations **can only be executed via TPI**, integrate `tpi_exec` with `sudoers`.

### 9.1 Example sudoers Policy

Create a dedicated sudoers file:

```bash
visudo -f /etc/sudoers.d/tpi
```

Example policy:

```sudoers
# Allow admins to run TPI wrapper
Cmnd_Alias TPI_WRAPPER = /usr/local/sbin/tpi_exec *

# Critical commands (must NOT be run directly)
Cmnd_Alias CRITICAL_CMDS = \
  /usr/bin/systemctl restart dirsrv@*, \
  /usr/bin/systemctl stop dirsrv@*, \
  /usr/sbin/firewall-cmd *, \
  /usr/sbin/setenforce *, \
  /usr/bin/vi /etc/selinux/config

# Admin group may run wrapper without password
%admins ALL=(root) NOPASSWD: TPI_WRAPPER

# Explicitly deny direct execution of critical commands
%admins ALL=(root) !CRITICAL_CMDS
```

Result:
- Admins **must** use `tpi_exec`
- Direct execution is blocked
- Enforcement is centralized and auditable

---

## 10. Journald and auditd Integration

### 10.1 Journald Logging

Extend `tpi_exec` to log approvals and executions:

```bash
logger -t tpi_exec "Approval recorded by $USER for: $CMD"
```

Example insertion points:
- On approval record
- On execution
- On timeout expiration

Logs can be queried with:

```bash
journalctl -t tpi_exec
```

---

### 10.2 auditd Integration

Add an audit rule to track TPI usage:

```bash
cat > /etc/audit/rules.d/tpi.rules <<'EOF'
-w /usr/local/sbin/tpi_exec -p x -k tpi-exec
-w /var/lock/tpi_action.lock -p wa -k tpi-lock
EOF

augenrules --load
```

Query audit trail:

```bash
ausearch -k tpi-exec
ausearch -k tpi-lock
```

---

## 11. Policy-Driven TPI Framework (/etc/tpi.d)

To scale beyond a single wrapper, TPI can be converted into a **policy-driven framework**.

### 11.1 Directory Structure

```text
/etc/tpi.d/
├── systemctl.conf
├── firewall.conf
├── selinux.conf
```

Each file defines:
- Allowed commands
- Required approver count
- Timeout

---

### 11.2 Example Policy File

`/etc/tpi.d/systemctl.conf`

```ini
REQUIRED=3
TIMEOUT=600
ALLOW="^systemctl (restart|stop) dirsrv@.*$"
```

---

### 11.3 Wrapper Logic (Conceptual)

`tpi_exec` evaluates the command against policies:

1. Match command regex
2. Load REQUIRED/TIMEOUT
3. Enforce quorum
4. Execute or reject

This allows:
- Per-command quorum
- Safer extensibility
- Cleaner audits

---

## 12. Threat Model (Auditor-Focused)

### 12.1 Threats Addressed

| Threat | Mitigation |
|------|------------|
| Single-admin misuse | Multi-person approval required |
| Accidental execution | Explicit re-run confirmation |
| Privilege escalation | sudoers command restriction |
| Insider risk (solo) | Quorum enforcement |
| Script tampering | Root-owned, audited path |

---

### 12.2 Threats Not Addressed

- Collusion among approvers
- Compromised root account
- Host-level kernel compromise
- Centralized approval workflows

These require enterprise PAM or hardware-backed controls.

---

## 13. Summary

This enhanced TPI framework provides:

- Enforced separation-of-duties
- Command-level control via sudoers
- Full audit visibility (journald + auditd)
- Extensible, policy-driven approvals
- Clear auditor-ready threat boundaries

It is a **practical, transparent control** for SLES systems where simplicity and trust boundaries matter.

---

**End of Document**

