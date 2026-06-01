# RPI-Hardening-Playbook
# 🔒 Raspberry Pi Security Hardening

[![Version](https://img.shields.io/badge/version-1.3.0-blue.svg)](CHANGELOG.md)
[![Ansible](https://img.shields.io/badge/ansible-2.14%2B-red.svg)](https://docs.ansible.com)
[![Platform](https://img.shields.io/badge/platform-Raspberry%20Pi%20OS%20Bookworm-c51a4a.svg)](https://www.raspberrypi.com/software/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

An Ansible playbook that takes a stock Raspberry Pi OS (Bookworm) installation
and applies a comprehensive set of security hardening controls, based on the
[Raspberry Pi Security Hardening Complete Guide](https://ohyaan.github.io/tips/raspberry_pi_security_hardening_complete_guide/).

Every section is independently toggleable. Before and after configuration
reports are written to the Pi automatically on each run.

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
rpi-security-hardening/
├── rpi_harden.yml              # Main playbook — all hardening logic
├── inventory.ini.example       # Inventory template — copy and edit
├── templates/
│   ├── iptables-rules.v4.j2   # iptables rate-limiting rules (Jinja2)
│   └── jail.local.j2          # fail2ban jail configuration (Jinja2)
├── CHANGELOG.md                # Full version history
├── LICENSE                     # MIT licence
└── README.md                   # This file
```

---

## Prerequisites

### 1. Ansible Control Machine (your laptop or desktop)

Requires Python 3.9+ and Ansible 2.14+.

```bash
pip3 install ansible

ansible-galaxy collection install ansible.posix community.general

ansible --version   # confirm 2.14+
```

### 2. Raspberry Pi — stock Raspberry Pi OS Bookworm

- Pi is powered on and reachable on the network
- SSH is enabled (Raspberry Pi Imager → Advanced Options, or `sudo raspi-config` → Interface Options → SSH)
- Default user `pi` exists with password authentication working
- Pi has internet access (apt packages are installed during the run)

### 3. SSH key pair on the control machine

The playbook deploys your public key to the Pi **before** disabling password
auth. Generate a dedicated key if you don't have one:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/pi_hardening_key
cat ~/.ssh/pi_hardening_key.pub   # copy this — you'll need it in step 4
```

### 4. Python on the Pi

Raspberry Pi OS Bookworm ships with Python 3 — confirm with:

```bash
ssh pi@<pi-ip> "python3 --version"
```

---

## Quickstart

### 1. Clone the repo

```bash
git clone https://github.com/YOUR_USERNAME/rpi-security-hardening.git
cd rpi-security-hardening
```

### 2. Create your inventory file

```bash
cp inventory.ini.example inventory.ini
```

Edit `inventory.ini` and replace the IP address with your Pi's:

```ini
[raspberrypis]
192.168.1.100 ansible_user=pi ansible_port=22
```

### 3. Edit the configuration block in `rpi_harden.yml`

Open the file and fill in the `vars:` section at the top. The minimum
required changes are:

```yaml
current_ssh_port: 22              # Pi's current SSH port
current_ssh_user: "pi"            # Pi's current login user
ssh_new_port: 2222                # Port SSH will move to after hardening
ssh_public_key: "ssh-ed25519 AAAAC3..." # your full public key string
local_network_cidr: "192.168.1.0/24"    # your LAN subnet
alert_email: "you@example.com"          # where reports and alerts go
```

### 4. Dry run (recommended before first full run)

```bash
ansible-playbook -i inventory.ini rpi_harden.yml \
  --ask-become-pass --check --diff
```

`--check` simulates all changes without applying them. `--diff` shows exactly
what would change in every config file.

### 5. Full hardening run

```bash
ansible-playbook -i inventory.ini rpi_harden.yml --ask-become-pass
```

### 6. After the run — update your inventory

The SSH port changes from 22 to `ssh_new_port` (default 2222). Update
`inventory.ini` for future runs:

```ini
[raspberrypis]
192.168.1.100 ansible_user=pi ansible_port=2222
```

Connect using your key:

```bash
ssh -i ~/.ssh/pi_hardening_key -p 2222 pi@192.168.1.100
```

---

## Section Toggles

Every hardening section can be independently enabled or disabled in the
`vars:` block near the top of `rpi_harden.yml`:

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

Set any value to `false` to skip that section entirely.

You can also run individual sections using Ansible tags:

```bash
# SSH hardening only
ansible-playbook -i inventory.ini rpi_harden.yml --tags ssh --ask-become-pass

# Firewall only
ansible-playbook -i inventory.ini rpi_harden.yml --tags firewall --ask-become-pass

# Reports only (no changes made)
ansible-playbook -i inventory.ini rpi_harden.yml --tags report --ask-become-pass
```

Available tags: `report`, `harden`, `updates`, `users`, `ssh`, `firewall`,
`iptables`, `fail2ban`, `services`, `filesystem`, `kernel`, `apparmor`,
`hardware`, `aide`, `logwatch`, `tripwire`, `monitoring`, `log2ram`, `vim`

---

## Before / After Reports

On every run the playbook writes two reports to `/var/log/security-hardening/`
on the Pi:

| File | Contents |
|---|---|
| `pre-hardening-report.txt` | Snapshot of services, ports, sshd config, sysctl, fstab, users — captured before any changes |
| `post-hardening-report.txt` | Same snapshot after all changes, plus a summary of which sections were enabled |

Pull them to your local machine:

```bash
scp -P 2222 -i ~/.ssh/pi_hardening_key \
  pi@192.168.1.100:/var/log/security-hardening/*.txt .
```

> **Note:** If `do_log2ram: true`, reports live inside the RAM-backed `/var/log`
> and are synced to SD on shutdown/reboot. They survive reboots but not sudden
> power loss. To make reports always persistent, set `report_dir` in the config
> to a path outside `/var/log`, such as `/home/pi/security-reports`.

---

## Important Caveats

### SSH lockout risk
The playbook deploys your public key **before** disabling password auth. If
`ssh_public_key` is empty the playbook aborts immediately. Always test that
your key works before a full run:

```bash
ssh -i ~/.ssh/pi_hardening_key pi@<pi-ip>
```

### `chattr +i` makes files immutable
When `do_file_attributes: true`, `/etc/passwd`, `/etc/shadow`, and `/etc/group`
become immutable — even root cannot modify them without removing the attribute
first. If you need to add users after hardening:

```bash
sudo chattr -i /etc/passwd /etc/shadow /etc/group
# make your changes
sudo chattr +i /etc/passwd /etc/shadow /etc/group
```

Consider `do_file_attributes: false` on systems still being provisioned.

### Sections that require a reboot
- **AppArmor** — kernel enforcement via cmdline.txt takes effect after reboot
- **Hardware disable** — config.txt changes take effect after reboot
- **log2ram** — `/var/log` is not moved to RAM until after reboot

The playbook does not reboot automatically.

### log2ram sizing
The RAM allocation must exceed your current log directory size. Check first:

```bash
du -sh /var/log
```

Recommended values for `log2ram_size`:
- `40M` — minimal headless Pi, very few services
- `100M` — typical hardened Pi with fail2ban, AIDE, logwatch *(default)*
- `200M` — many services or verbose logging

### Tripwire is install-only
`do_tripwire: true` only installs the package. Key setup and database
initialisation must be completed manually after the run:

```bash
sudo tripwire-setup-keyfiles
sudo tripwire --init
```

### AIDE is slow on first run
Building the AIDE database scans the entire filesystem and takes 10–20 minutes
on a Raspberry Pi. This is expected.

### Email alerts require a mail relay
The daily security script, AIDE check, and port scan detection all send alerts
via `mail`. Configure a relay on the Pi if you need email delivery:

```bash
sudo apt install ssmtp mailutils
sudo nano /etc/ssmtp/ssmtp.conf
```

---

## Recovery / Revert

### Locked out of SSH
Connect a keyboard and monitor directly, then:

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
sudo iptables -F
sudo iptables -X
sudo iptables -P INPUT ACCEPT
```

---

## Contributing

Pull requests are welcome. Please:

1. Fork the repo and create a feature branch (`git checkout -b feature/my-change`)
2. Make your changes in `rpi_harden.yml`
3. Run a syntax check: `ansible-playbook --syntax-check -i inventory.ini rpi_harden.yml`
4. Update `CHANGELOG.md` with the new version following [Keep a Changelog](https://keepachangelog.com) format
5. Bump the version number in:
   - The `CHANGELOG.md` header
   - The `# Version :` comment at the top of `rpi_harden.yml`
   - The `[![Version]` badge in `README.md`
6. Open a pull request against `main`

See [CHANGELOG.md](CHANGELOG.md) for the full version history.

---

## Versioning

This project follows [Semantic Versioning](https://semver.org):

- **MAJOR** (`X.0.0`) — breaking changes (e.g. renamed variables, restructured inventory)
- **MINOR** (`x.X.0`) — new hardening sections or features added in a backwards-compatible way
- **PATCH** (`x.x.X`) — bug fixes, documentation corrections, minor tweaks to existing tasks

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
