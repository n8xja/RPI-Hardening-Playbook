# Changelog

All notable changes to this project are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.3.0] — 2025-01-01

### Added
- **VIM** — new section `do_vim` installs vim, registers it as the
  system-wide default editor via `update-alternatives`, sets `EDITOR` and
  `VISUAL` in `/etc/environment`, and deploys `/etc/vim/vimrc.local` with
  mouse tracking disabled (`set mouse=`) plus sensible server defaults.

---

## [1.2.0] — 2025-01-01

### Added
- **log2ram** — new section `do_log2ram` installs log2ram from the official
  azlux.fr apt repository, configuring the RAM disk size, mail alerting, and
  rsync sync method via dedicated `log2ram_*` variables in the config block.
- `log2ram_size` variable — RAM to dedicate to `/var/log` (default `100M`).
- `log2ram_mail` variable — enable/disable low-disk email alerts.
- `log2ram_use_rsync` variable — use rsync for sync-to-disk (recommended).
- Post-hardening report now includes log2ram service status and config file.
- README note on report persistence when log2ram is active.

---

## [1.1.0] — 2025-01-01

### Added
- **System updates** (`do_system_updates`, `do_auto_updates`) — apt upgrade
  and unattended-upgrades with automatic reboot disabled.
- **Password policy** (`do_password_policy`) — libpam-pwquality with 12-char
  minimum, mixed case, digits, special characters, dictionary check.
- **User account hardening** (`do_disable_unused_accounts`) — lock/disable
  shell for games, news, uucp system accounts. Optional dedicated admin user
  creation with `create_admin_user`, `admin_username`, `admin_password`.
- **SSH hardening** (`do_ssh_hardening`) — public key deployment, password
  auth disabled, configurable port, root login disabled, connection limits,
  X11/TCP forwarding disabled, hardened cipher/MAC/KEX suites.
- **UFW firewall** (`do_ufw_firewall`) — default-deny incoming, SSH on custom
  port, HTTP/HTTPS restricted to local LAN, extensible via `ufw_extra_rules`.
- **iptables rate limiting** (`do_iptables_rules`) — SSH connection rate
  limiting and SYN flood protection via iptables-persistent.
- **fail2ban** (`do_fail2ban`) — SSH jail with configurable bantime, findtime,
  maxretry; rendered from Jinja2 template against active SSH port variable.
- **Service hardening** (`do_disable_services`) — stop and disable bluetooth,
  avahi-daemon, triggerhappy; list configurable via `services_to_disable`.
- **File permissions** (`do_file_permissions`) — enforce correct permissions
  on /etc/passwd, shadow, gshadow, group, and .ssh directories.
- **File attributes** (`do_file_attributes`) — `chattr +i` immutability on
  /etc/passwd, /etc/shadow, /etc/group.
- **Mount options** (`do_mount_options`) — tmpfs for /tmp and /var/tmp with
  nosuid, nodev, noexec, relatime.
- **Kernel hardening** (`do_kernel_hardening`) — sysctl parameters for
  network security (redirect, source route, martian logging, syncookies) and
  memory protection (ASLR).
- **AppArmor** (`do_apparmor`) — install apparmor + apparmor-utils, enable
  at boot via kernel cmdline, start service.
- **Hardware disable** (`do_hardware_disable`) — optional disable of
  Bluetooth, WiFi, audio, camera via /boot/firmware/config.txt overlays;
  independently controlled via `hw_disable_*` variables.
- **AIDE** (`do_aide`) — install, initialise database, daily cron check
  script with email alert on changes.
- **LogWatch** (`do_logwatch`) — install and configure for daily digest
  covering sshd, kernel, fail2ban services.
- **Tripwire** (`do_tripwire`) — install only (interactive key setup must
  be completed manually).
- **Daily security script** (`do_daily_security_script`) — cron job
  reporting failed logins, current users, open ports, firewall status.
- **Port scan detection** (`do_port_scan_detection`) — shell script deployed
  as a systemd service monitoring kern.log for SYN flood events.
- Jinja2 templates for iptables rules and fail2ban jail.local.
- `inventory.ini.example` template.
- `local_network_cidr` and `ufw_extra_rules` firewall configuration variables.
- `ssh_harden_ciphers` toggle for crypto hardening.
- `alert_email` variable used across AIDE, logwatch, daily scripts.

---

## [1.0.0] — 2025-01-01

### Added
- Initial playbook structure with `hosts: raspberrypis` and full `vars:` 
  configuration block.
- Pre-hardening report — collects OS info, users, enabled services, open
  ports, UFW status, sshd config, sysctl values, AppArmor status, fail2ban
  status, fstab, written to `{{ report_dir }}/pre-hardening-report.txt`.
- Post-hardening report — same collection after all tasks, with section
  toggle summary, written to `{{ report_dir }}/post-hardening-report.txt`.
- Section toggle framework — every hardening section gated by a `do_*`
  boolean variable; all tasks use `when:` conditions.
- `current_ssh_port` and `current_ssh_user` variables to reflect the Pi's
  pre-hardening state for Ansible connection.
- Playbook aborts immediately if `ssh_public_key` is empty.
- `report_dir` variable for configurable report output path.
- Ansible tags: `report`, `harden`, and per-section tags.

---

[1.3.0]: https://github.com/YOUR_USERNAME/rpi-security-hardening/compare/v1.2.0...v1.3.0
[1.2.0]: https://github.com/YOUR_USERNAME/rpi-security-hardening/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/YOUR_USERNAME/rpi-security-hardening/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/YOUR_USERNAME/rpi-security-hardening/releases/tag/v1.0.0
