# ⚔️ Arch Linux — Post-Install Security Ritual

Linux has a reputation for being secure — but a fresh Arch install is not a hardened one.  

> Follow these 8 steps every time you spin up a new machine.

---

## 1. Users, Groups & Sudo

The principle: **never operate as root**. Create a non-privileged user and grant only the access it needs.

```bash
# Create a new user with a home directory
useradd -m -G wheel -s /bin/bash <username>
passwd <username>
```

```bash
# Enable sudo for the wheel group
# add this line in .bashrc -
# export SUDO_EDITOR=nvim
visudo
# Uncomment: %wheel ALL=(ALL:ALL) ALL
```

**Key points:**
- Add the user to `wheel` (and other groups as needed: `audio`, `video`, `storage`, `network`).
- Verify group membership with `groups <username>`.
- Prefer `sudo` for one-off privileged commands, not a persistent root shell.
- Do not add users to unnecessary groups — every extra group is expanded attack surface.

```bash
# Check which groups your user belongs to
id <username>
```

---

## 2. File Permissions

Understand and audit permissions before trusting any file on the system.

```bash
# Read (r=4), Write (w=2), Execute (x=1)
# Format: [owner][group][others]
# Example: 755 = rwxr-xr-x

ls -la /path/to/file
stat /path/to/file
```

**Common permission targets to audit:**

| Path | Recommended | Reason |
|---|---|---|
| `/etc/passwd` | `644` | World-readable, no write |
| `/etc/shadow` | `640` | Root + shadow group only |
| `/etc/sudoers` | `440` | Read-only, never world-writable |
| `~/.ssh/` | `700` | Private to your user |
| `~/.ssh/authorized_keys` | `600` | Private to your user |

```bash
# Fix permissions on home directory
chmod 700 ~
chmod 600 ~/.ssh/authorized_keys
```

**SUID/SGID audit — find unexpected elevated binaries:**
```bash
find / -perm /4000 -type f 2>/dev/null   # SUID
find / -perm /2000 -type f 2>/dev/null   # SGID
```

---

## 3. Lock the Root Account

Root should be inaccessible directly. Force all privileged access through `sudo`.

```bash
# Lock the root password (disables direct root login)
passwd -l root

# Verify — the hash will be prefixed with '!'
grep root /etc/shadow
```

```bash
# Also disable root login via PAM (optional belt-and-suspenders)
# In /etc/pam.d/su, uncomment:
# auth required pam_wheel.so use_uid
```

> **Why?** Even if an attacker gets root's password hash, a locked account cannot be authenticated into directly. All privilege escalation is now logged through `sudo`.

---

## 4. Firewall with UFW

A fresh Arch install has **no firewall rules**. Fix that immediately.

```bash
# Install UFW
sudo pacman -S ufw

# Set sane defaults: deny all incoming, allow all outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH if you use it (add BEFORE enabling!)
sudo ufw allow ssh       # or: sudo ufw allow 22/tcp

# Enable UFW and set it to start on boot
sudo ufw enable
sudo systemctl enable ufw

# Check status
sudo ufw status verbose
```

**Common rule examples:**
```bash
sudo ufw allow 80/tcp        # HTTP
sudo ufw allow 443/tcp       # HTTPS
sudo ufw allow from 192.168.1.0/24   # LAN only
sudo ufw deny 23/tcp         # Explicitly block Telnet
```

> **Tip:** Always add `allow ssh` before enabling if you're on a remote machine — locking yourself out is a real risk.

---

## 5. SSH Hardening

If SSH is enabled, it is a potential attack vector. Tighten it down.

```bash
sudo vim /etc/ssh/sshd_config
```

**Apply these settings:**

```ini
# Disable root login entirely
PermitRootLogin no

# Disable password authentication — use keys only
PasswordAuthentication no
ChallengeResponseAuthentication no

# Limit to specific users
AllowUsers <username>

# Use a non-standard port (obscures from automated scanners)
Port 2222

# Disable X11 forwarding if not needed
X11Forwarding no

# Reduce login grace time
LoginGraceTime 30

# Limit auth attempts
MaxAuthTries 3
```

```bash
# Restart SSH after changes
sudo systemctl restart sshd

# Generate an SSH key pair on your CLIENT machine (not server)
ssh-keygen -t ed25519 -C "your@email.com"

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub <username>@<server>
```

> **Golden rule:** Disable `PasswordAuthentication` only **after** confirming key-based login works.

---

## 6. System Maintenance (Pacman & Paru)

A secure system is a **maintained** system. Stale packages are the most common real-world attack vector.

```bash
# Full system upgrade
sudo pacman -Syu

# Install paru (AUR helper — builds in a clean chroot)
sudo pacman -S --needed base-devel git
git clone https://aur.archlinux.org/paru.git
cd paru && makepkg -si

# Update everything including AUR packages
paru -Syu
```

**Maintenance habits:**
```bash
# Remove orphaned packages
sudo pacman -Rns $(pacman -Qtdq)

# Clear package cache (keep last 3 versions)
sudo paccache -r

# Check for packages not in any repo (foreign / AUR)
pacman -Qm

# Audit packages with known CVEs
sudo pacman -S arch-audit
arch-audit
```

**Automate updates with a systemd timer (optional):**
```bash
sudo systemctl enable --now pacman-filesdb-upgrade.timer
```

> **AUR Warning:** Always inspect `PKGBUILD` files before building. AUR is community-maintained — trust, but verify.

---

## 7. Debugging & Logs

Know how to read your system. Logs are your first line of diagnosis for anything suspicious.

```bash
# View the systemd journal (all logs)
journalctl

# Follow live output
journalctl -f

# Show logs since last boot
journalctl -b

# Filter by service
journalctl -u sshd
journalctl -u NetworkManager

# Show only errors and above
journalctl -p err

# Show kernel messages
journalctl -k
# or
dmesg | less
```

**Check for failed services:**
```bash
systemctl --failed
```

**Security-relevant log targets:**
```bash
# Auth log — watch for failed logins, sudo use
journalctl _COMM=sudo
journalctl _COMM=sshd

# Last logins
last
lastb    # failed logins
who      # currently logged in users
```

> **Tip:** Pipe `journalctl` output to `grep` for quick pattern matching:  
> `journalctl -b | grep -i "fail\|error\|denied"`

---

## 8. Secure Boot with sbctl + GRUB

Secure Boot ensures only cryptographically signed bootloaders and kernels can run. Without it, physical access = game over.

```bash
# Install sbctl
sudo pacman -S sbctl

# Check Secure Boot status
sbctl status
# Should show: Setup Mode: Enabled (to enroll your own keys)
```

```bash
# Create your own signing keys
sudo sbctl create-keys

# Enroll your keys into firmware (Microsoft keys included for hardware compatibility)
sudo sbctl enroll-keys -m

# Sign your GRUB EFI binary
sudo sbctl sign -s /boot/EFI/GRUB/grubx64.efi

# Sign your kernel (if using a unified kernel image)
sudo sbctl sign -s /boot/vmlinuz-linux

# Verify signed files
sbctl verify
```

```bash
# Regenerate GRUB config after any changes
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Check sbctl status — all entries should show ✓
sbctl status
```

> **Prerequisites:**  
> - System must be in **Setup Mode** in UEFI firmware.  
> - GRUB must be installed in UEFI mode (not legacy BIOS).  
> - After enrolling keys, re-enable Secure Boot in your UEFI settings.

---

## Quick Checklist

Run through this after every fresh install:

- [ ] Non-root user created and added to `wheel`
- [ ] Root account locked (`passwd -l root`)
- [ ] File permissions on sensitive paths audited
- [ ] UFW installed, configured, and enabled
- [ ] SSH hardened: no root login, no passwords, keys only
- [ ] System fully upgraded (`paru -Syu`)
- [ ] Orphaned packages removed
- [ ] `journalctl` checked for errors post-setup
- [ ] Secure Boot keys enrolled and binaries signed
- [ ] Reboot and verify everything comes up clean

---

## References

- [The Rad Lectures – YouTube](https://www.youtube.com/watch?v=8Oz4CIB4YjU)
- [Arch Wiki – Security](https://wiki.archlinux.org/title/Security)
- [Arch Wiki – UFW](https://wiki.archlinux.org/title/Uncomplicated_Firewall)
- [Arch Wiki – SSH Keys](https://wiki.archlinux.org/title/SSH_keys)
- [Arch Wiki – Secure Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)
- [Arch Wiki – Pacman Tips](https://wiki.archlinux.org/title/Pacman/Tips_and_tricks)

---

*Last updated: June 2026 — Always cross-reference with the [Arch Wiki](https://wiki.archlinux.org) for the latest guidance.*
