# Proxmox OneDrive Backup

Automated backup sync from Proxmox VE to OneDrive using restic and resticprofile.

## Setup

```bash
# 1. Install dependencies
# Install rclone
sudo apt install rclone
# Or install latest version via script:
# curl https://rclone.org/install.sh | sudo bash

# Install restic
RESTIC_VERSION=$(curl -s https://api.github.com/repos/restic/restic/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
wget -q "https://github.com/restic/restic/releases/download/${RESTIC_VERSION}/restic_${RESTIC_VERSION#v}_linux_amd64.bz2" -O /tmp/restic.bz2
bunzip2 /tmp/restic.bz2 && chmod +x /tmp/restic && sudo mv /tmp/restic /usr/local/bin/restic

RESTICPROFILE_VERSION=$(curl -s https://api.github.com/repos/creativeprojects/resticprofile/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
wget -q "https://github.com/creativeprojects/resticprofile/releases/download/${RESTICPROFILE_VERSION}/resticprofile_${RESTICPROFILE_VERSION#v}_linux_amd64.tar.gz" -O /tmp/resticprofile.tar.gz
tar -xzf /tmp/resticprofile.tar.gz -C /tmp && chmod +x /tmp/resticprofile && sudo mv /tmp/resticprofile /usr/local/bin/resticprofile

# 2. Configure OneDrive remote
rclone config
# Name: onedrive
# Type: onedrive
# Account: myaccount@example.com

# Copy rclone config to /root for system backups (requires sudo)
sudo mkdir -p /root/.config/rclone
sudo cp ~/.config/rclone/rclone.conf /root/.config/rclone/rclone.conf
sudo chmod 600 /root/.config/rclone/rclone.conf

# 3. Generate restic password
openssl rand -base64 32 > password
chmod 600 password

# 4. Install configuration files
mkdir -p logs

# 5. Initialize repositories
RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c profiles.yaml vm init
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml system init

# 6. Install schedules
sudo resticprofile -c /home/serveradmin/pve-backup/profiles.yaml schedule

# 7. Install logrotate configuration
sudo cp logrotate.conf /etc/logrotate.d/resticprofile
```

## Configuration

- **VM Profile**: Backs up both VM and CT files from `/var/lib/vz/dump`
  - Repository: `rclone:onedrive:B/pve/meatball/vm`
  - Tags: `proxmox`, `vm`, `ct`
  - Schedule: Hourly backups, weekly forget/prune/check (Monday 10:00 AM)

- **System Profile**: Backs up system configuration files
  - Repository: `rclone:onedrive:B/pve/meatball/system`
  - Sources: `/etc`, `/root`, `/home` (excludes `*.iso` files)
  - Tags: `proxmox`, `system`
  - Schedule: Daily backups, forget/prune/check
  - **Requires sudo** for accessing `/etc`, `/root`, and `/home`

- **Config**: `profiles.yaml` (in current directory)
- **Password**: `password` (in current directory)
- **Rclone config**: `/root/.config/rclone/rclone.conf` (for system backups with sudo)
- **Logs**: `logs/*.log` (rotated monthly, 12 months retention)
- **Cache**: `/var/cache/restic` (global, shared by all profiles)

## Commands

### VM Profile (no sudo required)
```bash
# Manual backup
RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c profiles.yaml vm backup

# List snapshots
RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c profiles.yaml vm snapshots

# Check repository
RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c profiles.yaml vm check

# Apply retention policy
RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c profiles.yaml vm forget

# Prune repository
RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c profiles.yaml vm prune
```

### System Profile (sudo required)
```bash
# Manual backup (requires sudo for /etc, /root, /home)
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml system backup

# List snapshots
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml system snapshots

# Check repository
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml system check

# Apply retention policy
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml system forget

# Prune repository
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml system prune
```

### All Profiles
```bash
# Backup all profiles
RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c profiles.yaml all backup
# Note: System profile will still require sudo when scheduled

# List all snapshots
RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c profiles.yaml all snapshots
```

## Self-Update

Update tools to latest versions:

```bash
# Update rclone
sudo rclone selfupdate

# Update restic (if installed via binary)
sudo restic self-update

# Update resticprofile
sudo resticprofile self-update
```

Note: If tools were installed via package manager, use `apt upgrade` instead.

## Notes

- Uses restic for deduplication and encryption
- Retention policy: keep last 1, daily 7, weekly 4, monthly 12
- VM/CT backups run hourly, forget/prune/check run weekly on Monday at 10:00 AM
- System backups run daily, forget/prune/check also run daily
- Files deleted locally are preserved on OneDrive via restic retention
- Proxmox handles local backup retention automatically
- **System profile requires sudo** for accessing `/etc`, `/root`, and `/home` directories
- Log rotation: Logs are rotated monthly via logrotate (see `logrotate.conf`), keeping 12 months of compressed logs
- Healthcheck.io monitoring integrated for all backup, prune, and check operations
