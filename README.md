# VPS Ansible

Ansible playbook to provision and maintain a Linux server for deploying containerized apps.

Based on [Kamal Ansible Manager](https://github.com/guillaumebriday/kamal-ansible-manager) by [Guillaume Briday](https://github.com/guillaumebriday).

## Compatibility

**Operating Systems:**
- Ubuntu (20.04+)
- Debian (11+)

**Architectures:**
- AMD64 / x86_64
- ARM64 / aarch64

## What it does

- **packages**: Installs essential packages (curl, git, htop, fail2ban, ufw, etc.), enables unattended upgrades, removes snap
- **docker**: Installs Docker CE from official repo
- **firewall**: Configures UFW (deny incoming, allow outgoing, rate-limit SSH, allow HTTP/HTTPS)
- **security**: Hardens SSH config (no password auth, no root password login), configures auto-reboot for security updates
- **zram**: Configures compressed swap in RAM (zstd, 50% of RAM)
- **tmpfs**: Disables tmpfs on /tmp to avoid ENOSPC on RAM-constrained servers
- **reboot_if_needed**: Reboots if kernel updates require it. Automatically detects encrypted disks (dropbear) and handles LUKS unlock with passphrase prompt and retry

## Prerequisites

**macOS (Homebrew):**
```bash
brew install ansible
```

**Ubuntu/Debian:**
```bash
sudo apt update && sudo apt install ansible
```

**pip (any platform):**
```bash
pip install ansible
```

## Standalone usage

1. Clone this repository:
```bash
git clone https://github.com/fcatuhe/vps-ansible.git
cd vps-ansible
```

2. Create an `inventory` file with your server IP:
```ini
[webservers]
your.server.ip.address
```

3. Install Ansible Galaxy requirements:
```bash
ansible-galaxy install -r requirements.yml
```

4. Run the playbook:
```bash
ansible-playbook playbook.yml
```

## Usage as a git submodule

This repo can be used as a shared infrastructure submodule across multiple projects:

```bash
cd your-project
git submodule add https://github.com/fcatuhe/vps-ansible.git .infra/vps-ansible
```

Then create a project-level `playbook.yml` in `.infra/` that references the shared roles:

```yaml
---
- name: Provisioning webservers group
  hosts: webservers
  strategy: free
  vars:
    security_autoupdate_reboot: "false"
    security_autoupdate_reboot_time: "03:00"
    # Optional: healthcheck URL checked after encrypted reboot
    # healthcheck_url: "https://example.com/up"
  roles:
    - vps-ansible/roles/packages
    - vps-ansible/roles/docker
    - vps-ansible/roles/firewall
    - vps-ansible/roles/security
    - vps-ansible/roles/zram
    - vps-ansible/roles/tmpfs
    - vps-ansible/roles/reboot_if_needed
```

Project-level files (`.infra/`):
- `playbook.yml` — role selection and vars (committed)
- `ansible.cfg` — inventory path, defaults (committed)
- `inventory` — server IPs (git-ignored)

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `security_autoupdate_reboot` | `"false"` | Auto-reboot for security updates |
| `security_autoupdate_reboot_time` | `"03:00"` | Reboot time if enabled (24h format) |
| `healthcheck_url` | _(undefined)_ | URL to check after encrypted reboot |

### Encrypted disk (dropbear) support

The `reboot_if_needed` role automatically detects if dropbear is installed (`/etc/dropbear/initramfs`). If present, it:
1. Prompts for the LUKS passphrase
2. Reboots the server
3. Waits for dropbear on port 2222
4. Unlocks the disk (with retry on wrong passphrase)
5. Waits for SSH on port 22
6. Optionally checks `healthcheck_url`

No configuration needed — detection is automatic.

## License

MIT
