# linux-firewall-hardening

Part of a hands-on roadmap to practicing my Network Security Engineering skills. This project focuses on hardening a Linux server: SSH lockdown, host-based firewall rules with UFW, and intrusion prevention with fail2ban.

## Overview

Starting from a fresh Ubuntu Server 22.04 VM, this project walks through securing SSH access, enforcing firewall rules, and automatically banning brute-force login attempts — the baseline hardening steps expected on any internet-facing Linux host.

## Lab Architecture

- **Host:** Windows machine running Oracle VirtualBox
- **Guest:** Ubuntu Server 22.04 LTS (VirtualBox VM), NAT networking with port forwarding
- Host port 2222 forwarded to guest port 2222 (SSH)

## What Was Configured

**SSH Hardening**
- Key-based authentication only (`PasswordAuthentication no`)
- Root login disabled (`PermitRootLogin no`)
- Custom SSH port to reduce automated scan noise

**UFW (Uncomplicated Firewall)**
- Default deny incoming, allow outgoing
- Explicit allow rule for the custom SSH port only

**fail2ban**
- Monitors SSH auth attempts via the systemd journal
- Bans an IP after 3 failed attempts within 5 minutes, for 10 minutes

See `/configs` for the actual configuration files used (sanitized) and `/screenshots` for verification evidence.

## Troubleshooting Log

Two real issues came up during this build — documented here because working through them was as valuable as the config itself:

**1. Changing the SSH port had no effect**
After editing `sshd_config` and restarting the service, `ss -tlnp` still showed SSH listening on port 22. The cause: Ubuntu 22.04+ uses systemd **socket activation** for SSH (`ssh.socket`), which controls the listening port independently of `sshd_config`. Diagnosed by checking `systemctl status ssh` and noticing `TriggeredBy: ssh.socket`. Fixed by disabling the socket unit and letting `ssh.service` bind directly:
```bash
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
```
A stale process (same PID before and after the fix) also required an explicit `stop` + `start` rather than `restart`, since the running process had already inherited the old socket binding.

**2. fail2ban failed to start after editing jail.local**
`systemctl status fail2ban` showed `status=255/EXCEPTION`. Digging into `journalctl -xeu fail2ban.service` revealed the real error: `section 'sshd' already exists`. Copying `jail.conf` to `jail.local` and adding a new `[sshd]` block created a duplicate section, since the default template already includes one further down the file. Fixed by editing the existing `[sshd]` section in place instead of appending a second one.

## Reproduce This Build

1. Set up an Ubuntu Server 22.04 VM in VirtualBox with NAT + port forwarding
2. Enable key-based SSH and disable password auth (see `configs/sshd_config.sample`)
3. Install and configure UFW to deny by default, allow only your SSH port
4. Install fail2ban, configure the `[sshd]` jail using `configs/jail.local` as a reference
5. Test: attempt failed logins and confirm the IP gets banned via `fail2ban-client status sshd`

## Tools Used

VirtualBox, Ubuntu Server 22.04, UFW, fail2ban, OpenSSH

---

Part of a self-directed roadmap to build hands-on networking and cybersecurity skills. Read the full write-up: https://medium.com/@shittuadedeji10/hardening-a-linux-server-ssh-ufw-and-fail2ban-and-the-two-bugs-that-taught-me-the-most-697b3e47b2ed
