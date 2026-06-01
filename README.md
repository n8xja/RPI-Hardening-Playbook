# 🔒 Raspberry Pi Security Hardening

[![Version](https://img.shields.io/badge/version-1.4.0-blue.svg)](CHANGELOG.md)
[![Ansible](https://img.shields.io/badge/ansible-2.14%2B-red.svg)](https://docs.ansible.com)
[![Platform](https://img.shields.io/badge/platform-Raspberry%20Pi%20OS%20Bookworm-c51a4a.svg)](https://www.raspberrypi.com/software/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

An Ansible playbook that takes a stock Raspberry Pi OS (Bookworm) installation
and applies a comprehensive set of security hardening controls, based on the
[Raspberry Pi Security Hardening Complete Guide](https://ohyaan.github.io/tips/raspberry_pi_security_hardening_complete_guide/).

Supports two deployment modes — run from a **remote control machine** over SSH,
or run **locally on the Pi itself**. Every hardening section is independently
toggleable. Before and after configuration reports are written automatically on
each run.

---

## Features

| Section | What it does |
|---|---|
| System updates | Full apt upgrade + unattended-upgrades |
| Password policy | libpam-pwquality with strong defaults |
| User accounts | Disable/lock unused system accounts, optional admin user |
| SSH hardening | Key-only auth, port change, cipher hardening, connection limits |
| UFW firewall | Default-deny with allow rules for SSH and local LAN |
| iptables | Rate-limiting rules and SYN flood protection |
| fail2ban | SSH brute-force protection with custom jail |
| Service hardening | Disable bluetooth, avahi-daemon, triggerhappy |
| File permissions | Tighten /etc/passwd, shadow, group, .ssh |
| File attributes | `chattr +i` immutability on critical files |
| Mount options | nodev/nosuid/noexec on /tmp and /var/tmp |
| Kernel hardening | sysctl network and memory security parameters |
| AppArmor | Install, enable, and set enforcing mode |
| Hardware disable | Disable BT/WiFi/audio/camera via config.txt |
| AIDE | File integrity monitoring with daily cron check |
| LogWatch | Daily log summary emails |
| Tripwire | File integrity (install only — interactive setup required) |
| Daily security script | Cron job reporting failed logins, open ports, firewall status |
| Port scan detection | Systemd service watching for SYN flood events |
| log2ram | Mount /var/log as RAM disk to protect SD card from write wear |
| VIM | Install vim, set as system-wide default editor, disable mouse tracking |

---

## Repository Structure

```
RPI-Hardening-Playbook/
├── rpi_harden.yml              # Main playbook — all hardening logic
├── inventory.ini.example       # Remote inventory template — copy and edit
├── inventory.local.ini         # Local inventory — use when running on the Pi itself
├── templates/
│   ├── iptables-rules.v4.j2   # iptables rate-limiting rules (Jinja2)
│   └── jail.local.j2          # fail2ban jail configuration (Jinja2)
├── CHANGELOG.md                # Full version history
├── CONTRIBUTING.md             # Versioning and contribution workflow
├── LICENSE                     # MIT licence
└── README.md                   # This file
```

---

## Deployment Modes

This playbook supports two modes of execution. Choose the one that fits your
setup. The mode is controlled by a single variable in `rpi_harden.yml`:

```yaml
deployment_mode: "remote"   # change to "local" for on-Pi execution
```

---

### Remote Mode (default)

You run Ansible on a **separate machine** (laptop, desktop, another server)
and it connects to the Pi over SSH to apply changes.

```
[Your laptop] ──SSH──► [Raspberry Pi]
  runs ansible             target
```

**How to use:**

```bash
ansible-playbook -i inventory.ini rpi_harden.yml --ask-become-pass
```

**What remote mode does that local mode cannot:**

- Deploys your SSH public key to the Pi before disabling password auth — the
  safest sequence, as key access is confirmed before the old auth method is removed
- Updates the live Ansible SSH connection mid-play after changing the SSH port,
  so the playbook continues uninterrupted after the port changes
- Keeps playbook output and reports on your control machine as well as on the Pi
- Lets you do a dry run (`--check --diff`) without touching the Pi at all
- Allows you to harden multiple Pis in one run by adding them to `inventory.ini`

**Limitations of remote mode:**

- Requires a separate machine with Ansible installed
- Requires the Pi to be reachable over the network before running
- The `--ask-become-pass` prompt means it cannot be run fully unattended
  without additional credential setup (e.g. Ansible Vault or SSH agent)

---

### Local Mode

You clone the repo **directly onto the Pi** and run the playbook there.
Ansible uses a local connection — no SSH involved at all.

```
[Raspberry Pi]
  runs ansible + is the target
```

**How to use:**

1. On the Pi, install Ansible:
   ```bash
   sudo apt update && sudo apt install -y ansible
   ansible-galaxy collection install ansible.posix community.general
   ```

2. Clone the repo:
   ```bash
   git clone https://github.com/n8xja/RPI-Hardening-Playbook.git
   cd RPI-Hardening-Playbook
   ```

3. Edit `rpi_harden.yml` — set `deployment_mode: "local"` and configure
   all other variables as normal (SSH port, firewall rules, email, etc.)

4. Run as root:
   ```bash
   sudo ansible-playbook -i inventory.local.ini rpi_harden.yml
   ```

**What local mode does that remote mode cannot:**

- Works without a second machine — useful for one-off Pi setups or when
  you don't have another device running Ansible
- Works without any network connectivity to the Pi (useful for air-gapped
  or headless setups connected via keyboard/monitor)
- No SSH credential setup required — running as root handles privilege escalation directly

**Limitations of local mode:**

| Limitation | Detail |
|---|---|
| **No automatic SSH key deployment** | The `authorized_key` task is skipped. After the playbook runs and SSH password auth is disabled, you must add your public key manually *before* your next remote SSH session, or you will be locked out. See the warning below. |
| **No mid-play port update** | The task that updates Ansible's live SSH connection after changing the port is skipped (irrelevant in local mode, but means if you run the playbook a *second* time remotely without updating `inventory.ini`, it will fail to connect). |
| **Ansible runs on the Pi's RAM and SD card** | Installing Ansible consumes ~150MB of disk and adds some SD card writes during installation. Remove it after hardening if resource conservation matters: `sudo apt remove ansible`. |
| **Output stays on the Pi** | Playbook output and reports exist only on the Pi itself. Copy them off manually if needed. |
| **Single target only** | Local mode can only harden the machine it runs on. For multiple Pis, remote mode is the right choice. |
| **No safe dry run** | `--check --diff` works technically, but since Ansible is running on the same machine it's testing against, some checks interact with live state in ways that don't occur with a genuinely separate control machine. |

> ⚠️ **Critical local mode warning — SSH lockout risk**
>
> In local mode, the playbook hardens `sshd_config` (changes port, disables
> password auth) but does **not** add your public key to `authorized_keys`.
> If you disconnect and try to SSH back in, you will be locked out unless
> your key is already present.
>
> Before running in local mode with `do_ssh_hardening: true`, either:
> - Pre-place your public key: `echo "ssh-ed25519 AAAA..." >> ~/.ssh/authorized_keys`
> - Or disable SSH hardening: `do_ssh_hardening: false` and harden it manually after
>
> The playbook prints a reminder at the SSH section when running in local mode.

---

### Side-by-Side Comparison

| Capability | Remote | Local |
|---|:---:|:---:|
| Requires second machine | ✅ Yes | ❌ No |
| Requires network connectivity | ✅ Yes | ❌ No |
| Deploys SSH public key automatically | ✅ Yes | ❌ No |
| Updates SSH connection mid-play | ✅ Yes | ❌ No (skipped) |
| Harden multiple Pis in one run | ✅ Yes | ❌ No |
| Safe dry run (`--check --diff`) | ✅ Yes | ⚠️ Partial |
| Runs without Ansible on control machine | ❌ No | ✅ Yes |
| Works air-gapped / no network | ❌ No | ✅ Yes |
| Leaves no Ansible install on Pi | ✅ Yes | ⚠️ Optional cleanup |
| Before/after reports off-Pi | ✅ Yes | ❌ Manual copy |

---

## Prerequisites

### Remote mode prerequisites

**On your control machine:**

```bash
pip3 install ansible
ansible-galaxy collection install ansible.posix community.general
ansible --version   # confirm 2.14+
```

**On the Pi:**

- SSH enabled (Raspberry Pi Imager → Advanced Options, or `sudo raspi-config` → Interface Options → SSH)
- Default user `pi` with password auth working
- Internet access for apt packages
- Python 3 installed (ships with Pi OS — confirm with `python3 --version`)

**SSH key pair on the control machine:**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/pi_hardening_key
cat ~/.ssh/pi_hardening_key.pub   # copy this into ssh_public_key in the vars block
```

### Local mode prerequisites

**On the Pi:**

```bash
sudo apt update && sudo apt install -y ansible git
ansible-galaxy collection install ansible.posix community.general
```

No SSH key setup required before running. See the SSH lockout warning above
for what to do *before* SSH password auth is disabled.

---

## Quickstart

### Remote

```bash
# 1. Clone on your control machine
git clone https://github.com/n8xja/RPI-Hardening-Playbook.git
cd RPI-Hardening-Playbook

# 2. Create inventory
cp inventory.ini.example inventory.ini
# Edit inventory.ini — set your Pi's IP address

# 3. Edit rpi_harden.yml vars block:
#      deployment_mode: "remote"
#      ssh_public_key:  "ssh-ed25519 AAAA..."   (your public key)
#      local_network_cidr: "192.168.1.0/24"
#      alert_email: "you@example.com"

# 4. Dry run
ansible-playbook -i inventory.ini rpi_harden.yml --ask-become-pass --check --diff

# 5. Full run
ansible-playbook -i inventory.ini rpi_harden.yml --ask-become-pass
```

### Local (on the Pi)

```bash
# 1. Install Ansible on the Pi
sudo apt update && sudo apt install -y ansible git
ansible-galaxy collection install ansible.posix community.general

# 2. Clone the repo
git clone https://github.com/n8xja/RPI-Hardening-Playbook.git
cd RPI-Hardening-Playbook

# 3. Pre-place your SSH public key (BEFORE running if do_ssh_hardening: true)
mkdir -p ~/.ssh
echo "ssh-ed25519 AAAA..." >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys

# 4. Edit rpi_harden.yml vars block:
#      deployment_mode: "local"
#      local_network_cidr: "192.168.1.0/24"
#      alert_email: "you@example.com"
#    (ssh_public_key can be left empty in local mode)

# 5. Run
sudo ansible-playbook -i inventory.local.ini rpi_harden.yml
```

---

## Section Toggles

Every hardening section can be independently enabled or disabled in the
`vars:` block of `rpi_harden.yml`:

```yaml
do_system_updates:           true
do_auto_updates:             true
do_password_policy:          true
do_disable_unused_accounts:  true
do_ssh_hardening:            true
do_ufw_firewall:             true
do_iptables_rules:           true
do_fail2ban:                 true
do_disable_services:         true
do_file_permissions:         true
do_file_attributes:          true   # chattr +i — see caveats
do_mount_options:            true
do_kernel_hardening:         true
do_apparmor:                 true
do_hardware_disable:         false  # review hw_disable_* vars first
do_aide:                     true   # slow on first run (10–20 min)
do_logwatch:                 true
do_tripwire:                 false  # requires manual post-install steps
do_daily_security_script:    true
do_port_scan_detection:      true
do_log2ram:                  true
do_vim:                      true
```

You can also target individual sections with Ansible tags:

```bash
# SSH hardening only
ansible-playbook -i inventory.ini rpi_harden.yml --tags ssh --ask-become-pass

# Firewall only
ansible-playbook -i inventory.ini rpi_harden.yml --tags firewall --ask-become-pass

# Reports only — no changes made
ansible-playbook -i inventory.ini rpi_harden.yml --tags report --ask-become-pass
```

Available tags: `report`, `harden`, `updates`, `users`, `ssh`, `firewall`,
`iptables`, `fail2ban`, `services`, `filesystem`, `kernel`, `apparmor`,
`hardware`, `aide`, `logwatch`, `tripwire`, `monitoring`, `log2ram`, `vim`

---

## Before / After Reports

On every run the playbook writes two reports to `{{ report_dir }}`
(default `/var/log/security-hardening/`) on the Pi:

| File | Contents |
|---|---|
| `pre-hardening-report.txt` | Snapshot of services, ports, sshd config, sysctl, fstab, users — captured before any changes |
| `post-hardening-report.txt` | Same snapshot after all changes, plus a summary of which sections were enabled and the deployment mode used |

Pull them to your local machine (remote mode):

```bash
scp -P 2222 -i ~/.ssh/pi_hardening_key \
  pi@192.168.1.100:/var/log/security-hardening/*.txt .
```

> **Note:** If `do_log2ram: true`, reports live inside the RAM-backed `/var/log`
> and sync to SD on shutdown/reboot. They survive reboots but not sudden power
> loss. To make reports always persistent, set `report_dir` to a path outside
> `/var/log`, such as `/home/pi/security-reports`.

---

## Important Caveats

### SSH lockout risk (remote mode)
The playbook deploys your public key **before** disabling password auth.
If `ssh_public_key` is empty the playbook aborts. Always test that your key
works before a full run: `ssh -i ~/.ssh/pi_hardening_key pi@<pi-ip>`

### SSH lockout risk (local mode)
Key deployment is skipped. Manually place your public key in
`~/.ssh/authorized_keys` **before** running if `do_ssh_hardening: true`.

### `chattr +i` makes files immutable
When `do_file_attributes: true`, `/etc/passwd`, `/etc/shadow`, and `/etc/group`
become immutable. To modify them later:
```bash
sudo chattr -i /etc/passwd /etc/shadow /etc/group
# make changes
sudo chattr +i /etc/passwd /etc/shadow /etc/group
```
Consider `do_file_attributes: false` on systems still being provisioned.

### Sections requiring a reboot
- **AppArmor** — kernel enforcement via cmdline.txt takes effect after reboot
- **Hardware disable** — config.txt changes take effect after reboot
- **log2ram** — `/var/log` is not moved to RAM until after reboot

The playbook does not reboot automatically.

### log2ram sizing
Check your current log size before setting `log2ram_size`:
```bash
du -sh /var/log
```
- `40M` — minimal headless Pi
- `100M` — typical hardened Pi *(default)*
- `200M` — many services or verbose logging

### Tripwire is install-only
Complete setup manually after the run:
```bash
sudo tripwire-setup-keyfiles
sudo tripwire --init
```

### AIDE is slow on first run
10–20 minutes on a Raspberry Pi. Expected.

### Email alerts require a mail relay
```bash
sudo apt install ssmtp mailutils
sudo nano /etc/ssmtp/ssmtp.conf
```

---

## After the Run — Update inventory.ini (remote mode)

The SSH port changes to `ssh_new_port` (default 2222). Update for future runs:

```ini
[raspberrypis]
192.168.1.100 ansible_user=pi ansible_port=2222
```

Connect with your key:
```bash
ssh -i ~/.ssh/pi_hardening_key -p 2222 pi@192.168.1.100
```

---

## Recovery / Revert

### Locked out of SSH
Connect keyboard and monitor directly:
```bash
sudo cp /etc/ssh/sshd_config.backup /etc/ssh/sshd_config
sudo systemctl restart ssh
```

### Remove file immutability
```bash
sudo chattr -i /etc/passwd /etc/shadow /etc/group
```

### Reset firewall
```bash
sudo ufw disable
sudo iptables -F && sudo iptables -X && sudo iptables -P INPUT ACCEPT
```

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full versioning and PR workflow.

In brief:
1. Fork, create a branch
2. Make changes, run `ansible-playbook --syntax-check`
3. Bump version in `rpi_harden.yml`, `README.md`, and `CHANGELOG.md`
4. Open a PR against `main`

---

## Versioning

[Semantic Versioning](https://semver.org): `MAJOR.MINOR.PATCH`

| Increment | When |
|---|---|
| **MAJOR** | Breaking change — renamed variable, restructured inventory |
| **MINOR** | New hardening section or feature added |
| **PATCH** | Bug fix, doc correction, minor tweak |

---

## Reference

- [Raspberry Pi Security Hardening Complete Guide](https://ohyaan.github.io/tips/raspberry_pi_security_hardening_complete_guide/)
- [Ansible Documentation](https://docs.ansible.com)
- [log2ram by azlux](https://github.com/azlux/log2ram)
- [fail2ban Documentation](https://www.fail2ban.org/wiki/index.php/Main_Page)
- [UFW Documentation](https://help.ubuntu.com/community/UFW)
- [AIDE Manual](https://aide.github.io)
- [AppArmor Wiki](https://gitlab.com/apparmor/apparmor/-/wikis/home)

---

## License

MIT — see [LICENSE](LICENSE) for details.
