# Debian-based System Configuration Hardening (minCIS_deb-based)

This project provides a production-grade, highly modular, and idempotent Ansible playbook designed to automatically harden Debian-based Linux systems. It drastically reduces the attack surface, implements mandatory access controls, hardens the kernel and filesystem, and enforces strict authentication policies based on industry-standard security guidelines.

## 🚨 Caution(s) 🚨

Executing this playbook will apply deep system modifications that may interfere with existing services or standard system behaviors. Please note that this is exclusively a **remediation tool**, designed to be deployed only *after* a proper security audit has been performed.

- **Thoroughly Test Before Deployment:** It is strongly recommended to apply these changes in a sterile staging or testing environment prior to any production roll-out.
- **Assumes a Clean Installation:** These roles were specifically engineered and tested against a fresh installation of Debian. If you are applying them to a pre-existing environment, manually review the configurations beforehand to accommodate any site-specific operational requirements.

---

## 🏗 Architecture

The playbook follows a modular Role-Based architecture, making it easy to turn specific hardening features on or off as needed.

### Directory Structure

```text
minCIS_deb-based/
├── Debian_based_v2.md     # Comprehensive manual hardening guide
├── inventory.ini          # Server inventory targeting your environments
├── site.yml               # Master playbook defining role execution order
├── group_vars/
│   └── all.yml            # Global feature toggles for all hardening metrics
└── roles/                 # Modular hardening roles
    ├── access_control/    # AppArmor and Auditd enforcement
    ├── auth_ssh/          # SSH, PAM policies, Sudo restrictions, Banners
    ├── common_handlers/   # Global handlers (Reboot, SSH restart, GRUB)
    ├── filesystem/        # Secure mount options and FS blacklisting
    ├── kernel_boot/       # Kernel sysctl, coredumps, GRUB permissions
    ├── network_firewall/  # Network sysctl, UFW, exotic protocols removal
    └── package_mgmt/      # APT security and legacy service purging
```

### Execution Flow (`site.yml`)

The main playbook targets the `debian_servers` group and executes the following roles sequentially:

1. **`common_handlers`**: Consolidates global handlers (e.g., `Restart SSH`, `Reboot System`, `Update GRUB`) invoked by multiple roles.

2. **`filesystem`**:
   - Blacklists obsolete and high-risk filesystem modules (cramfs, jffs2, hfs, usb-storage, etc.).
   - Enforces strict mount options (`nodev`, `nosuid`, `noexec`) on sensitive partitions (`/tmp`, `/var`, `/home`, etc.).

3. **`package_mgmt`**:
   - Disables weak APT dependencies (`Recommends` and `Suggests`).
   - Secures APT repository file permissions.
   - Purges legacy network services, graphical interfaces (GDM3/X11), and unnecessary daemons.

4. **`access_control`**:
   - Installs and enforces **AppArmor** profiles.
   - Restricts unprivileged AppArmor namespaces.
   - Installs **Auditd** for extensive system event auditing.
   - Automatically injects AppArmor and Auditd kernel parameters into the GRUB bootloader.

5. **`kernel_boot`**:
   - Secures `/boot/grub/grub.cfg` permissions to prevent local tampering.
   - Applies strict Kernel Sysctl hardening (enables ASLR, restricts dmesg, protects symlinks/hardlinks).
   - Disables core dumps and removes debugging tools like prelink/apport.

6. **`network_firewall`**:
   - Blacklists exotic networking protocols (dccp, tipc, rds, sctp).
   - Secures the TCP/IP stack via Sysctl (disables IP forwarding, enables SYN cookies, logs martians).
   - Installs and configures **UFW** (Uncomplicated Firewall) with a default allow-out/deny-in/deny-routed policy.

7. **`auth_ssh`**:
   - Deploys legal warning banners (`/etc/issue`, `/etc/motd`).
   - Heavily restricts SSH daemon access (`PermitRootLogin no`, `PasswordAuthentication no`, `X11Forwarding no`).
   - Implements strict **PAM** password complexity and expiration rules.
   - Enables detailed Sudo auditing and secures `cron` directories.

---

## ⚙️ Configuration & Toggles

Configuration is managed globally through `group_vars/all.yml` and can be overridden per host group in `inventory.ini`. Every hardening metric corresponds to a toggle variable, allowing highly granular control:

```yaml
# Examples of toggles in group_vars/all.yml:
hardening_fs_disable_modules: true
hardening_apparmor_enable: true
hardening_ufw_enable: true
hardening_ssh_secure: true
hardening_pam_secure: true
```

If a configuration breaks an application, you can simply change its corresponding toggle to `false` and re-run the playbook to bypass that specific restriction without disabling the entire role.

---

## 🚀 Setup and Execution Instructions

> [!NOTE]
> For more in-depth information, architectural concepts, and manual step-by-step equivalents, please refer to the comprehensive [Debian_based_v2.md](Debian_based_v2.md) guide included in this repository.

### 1. Environment and Policy Customization

Before running the playbook, you must align the configuration with your specific environment and company policies:
- **Inventory Settings:** Edit the `inventory.ini` file to specify your target nodes. You can separate servers into logical groups (`[test_nodes]`, `[production_nodes]`) and apply variable overrides.
- **Legal Warning Banner:** Edit the `motd` and `issue` banner text within `roles/auth_ssh/tasks/main.yml` to match your company's official legal login warning policy.

### 2. Pre-flight Security Verification

> [!WARNING]
> **SSH Keys Required!** The `auth_ssh` role forcibly disables password-based authentication (`PasswordAuthentication no`) and root login (`PermitRootLogin no`). **You MUST ensure you have an SSH Key registered and authorized for your admin user on the target servers before running this playbook.** If you do not, you will lock yourself out permanently.
> 
> **Test Environment First!** These changes drastically limit system capabilities. Certain non-standard applications and legacy protocols will break. **Always apply the playbook to `[test_nodes]` first** so you can tweak the toggles in `group_vars/all.yml` before touching production.

### 3. Securing Credentials (Ansible Vault)

Your `inventory.ini` shouldn't store plain-text credentials (`ansible_ssh_pass`, `ansible_become_pass`). Use Ansible Vault to encrypt them.
Create an encrypted file to hold your sensitive variables:

```bash
ansible-vault create group_vars/all_secrets.yml
```

Inside, add your passwords:

```yaml
ansible_ssh_pass: "your_secure_password"
ansible_become_pass: "your_secure_password"
```

Remove the plain-text passwords from `inventory.ini`.

### 4. Running the Playbook

To deploy the hardening configuration across all Debian servers using your Vault password:

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

### 5. Post-Execution

Certain kernel parameters, GRUB changes, and filesystem mount configurations require a full reboot. After a successful Ansible run, it is highly recommended to restart the target servers. The playbook may attempt to orchestrate a reboot via handlers automatically if deep system modifications were detected.

---

***Authored by [Thedarknova]***
