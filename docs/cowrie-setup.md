# Cowrie SSH Honeypot Setup

This document covers the installation and configuration of Cowrie, a medium-interaction SSH honeypot, on Proxmox VE as an LXC container.

---

## Infrastructure

| Detail | Value |
|--------|-------|
| Container | LXC 108 (cowrie-01) |
| OS | Debian 12 |
| IP | 192.168.1.186 (DHCP) |
| RAM | 512MB |
| Disk | 8GB |
| Cowrie Version | 2.9.14 |

---

## How It Works

Cowrie listens on port 2222 and simulates a real SSH server. Attackers (or testers) who connect are dropped into a fake shell environment — all commands are emulated, no real system access is granted. Every login attempt, command, and session is logged in detail.

Cloudflare's role is DNS only — all traffic stays internal.

---

## Installation

### 1. Create LXC in Proxmox

- Template: Debian 12
- Hostname: cowrie-01
- RAM: 512MB, 1 CPU, 8GB disk
- Network: DHCP

### 2. Install Dependencies

```bash
apt update && apt upgrade -y
apt install -y python3-virtualenv libssl-dev libffi-dev build-essential libpython3-dev python3-minimal authbind git
```

### 3. Create Dedicated User

```bash
adduser --disabled-password cowrie
su - cowrie
```

### 4. Clone Repo and Set Up Virtual Environment

```bash
git clone https://github.com/cowrie/cowrie
cd cowrie
virtualenv --python=python3 cowrie-env
source cowrie-env/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

### 5. Install Cowrie in Editable Mode

```bash
pip install -e .
```

This keeps all data files (fake filesystem, etc.) in place from the git repo.

### 6. Configure

```bash
cp etc/cowrie.cfg.dist etc/cowrie.cfg
```

Edit `etc/cowrie.cfg` as needed. Default config listens on port 2222.

### 7. Start Cowrie

```bash
cowrie-env/bin/cowrie start
```

### 8. Verify

```bash
cowrie-env/bin/cowrie status
```

---

## Testing

From another machine on the network:

```bash
ssh root@192.168.1.186 -p 2222
```

Use any password — Cowrie accepts all credentials by default. Run commands like `ls`, `whoami`, `cat /etc/passwd` to confirm the fake shell is working.

**Important:** Do not use real credentials when testing. Cowrie logs all usernames and passwords in plaintext.

---

## Logs

```bash
# Text log
cat /home/cowrie/cowrie/var/log/cowrie/cowrie.log

# JSON log (for Wazuh/SIEM integration)
cat /home/cowrie/cowrie/var/log/cowrie/cowrie.json
```

Logs include: source IP, credentials attempted, commands run, session duration, and TTY recordings.

---

## Wazuh Integration

*Planned — feed Cowrie JSON logs into Wazuh for alerting and dashboards.*

---

## Notes

- Cowrie runs as a dedicated `cowrie` system user, not root.
- Default fake hostname is `svr04` — can be changed in `cowrie.cfg` under `[honeypot] hostname`.
- Port 2222 is the default. To attract real internet attackers, port forward port 22 on your router to 192.168.1.186:2222.
- All data files (fake filesystem, fake /etc/passwd, etc.) are included in the git repo under `cowrie/data/`.

---

## To-Do

- [ ] Set up Wazuh agent on cowrie-01
- [ ] Configure Wazuh to ingest Cowrie JSON logs
- [ ] Write custom Wazuh detection rules for brute force patterns
- [ ] Consider exposing to internet via port forward for real attacker data
- [ ] Set up systemd service for auto-start on boot
