# Proxmox OneDrive Backup

![meatball](https://healthchecks.io/badge/746c60b3-c4f2-44fd-9e7e-594d19/eUYth26p/meatball.svg)

Automated backup sync from Proxmox VE to OneDrive using **restic**, **resticprofile**, and **rclone**.

**Technologies:** `restic` `resticprofile` `rclone` `healthchecks.io`

**Monitoring:** All backup, prune, and check operations are monitored via [healthchecks.io](https://healthchecks.io) with automatic failure notifications.

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
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml vm init
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
  - Schedule: Daily backups at midnight, weekly forget/prune/check (Monday at midnight)

- **System Profile**: Backs up system configuration files
  - Repository: `rclone:onedrive:B/pve/meatball/system`
  - Sources: `/etc`, `/root`, `/home` (excludes `*.iso` files)
  - Tags: `proxmox`, `system`
  - Schedule: Daily backups, forget/prune/check
  - **Requires sudo** for accessing `/etc`, `/root`, and `/home`

- **Config**: `profiles.yaml` (in current directory)
- **Password**: `password` (in current directory)
- **Rclone config**: `/root/.config/rclone/rclone.conf` (requires sudo to access)
- **Logs**: `logs/*.log` (rotated monthly, 12 months retention)
- **Cache**: `/var/cache/restic` (global, shared by all profiles)

## Commands

### VM Profile
```bash
# Manual backup (sudo may be required to access /root/.config/rclone/rclone.conf)
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml vm backup

# List snapshots
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml vm snapshots

# Check repository
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml vm check

# Apply retention policy
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml vm forget

# Prune repository
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml vm prune
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
# Backup all profiles (sudo required to access /root/.config/rclone/rclone.conf)
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml all backup

# List all snapshots
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf resticprofile -c /home/serveradmin/pve-backup/profiles.yaml all snapshots
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

## Monitoring

This backup system uses [healthchecks.io](https://healthchecks.io) to monitor all backup operations:

- **Backup operations**: Monitored with start, success, and failure notifications
- **Prune operations**: Monitored with start, success, and failure notifications  
- **Check operations**: Monitored with start, success, and failure notifications
- **Failure notifications**: Include error details in the notification body

All healthcheck URLs are configured in `profiles.yaml` and automatically ping healthchecks.io before and after each operation.

## Notes

- Uses **restic** for deduplication and encryption
- Uses **rclone** to sync backups to OneDrive cloud storage
- Uses **resticprofile** for managing backup profiles and scheduling
- Retention policy: keep last 1, daily 7, weekly 4, monthly 12
- VM/CT backups run daily at midnight, forget/prune/check run weekly on Monday at midnight
- System backups run daily at midnight, forget/prune/check also run daily at midnight
- Files deleted locally are preserved on OneDrive via restic retention
- Proxmox handles local backup retention automatically
- **All profiles require sudo** to access `/root/.config/rclone/rclone.conf` for rclone authentication
- **System profile additionally requires sudo** for accessing `/etc`, `/root`, and `/home` directories
- Log rotation: Logs are rotated monthly via logrotate (see `logrotate.conf`), keeping 12 months of compressed logs

## Appendix: Triggering Scheduled Jobs for Testing

To manually trigger scheduled jobs without waiting for their scheduled time, use systemd to start the service units directly.

**Important:** Jobs must be triggered **sequentially** (one after another) to avoid restic repository locking issues. Each job must complete before starting the next one.

### List All Scheduled Jobs

```bash
sudo systemctl list-units --type=timer | grep restic
```

### Trigger Individual Jobs (Sequentially)

**VM Profile** - Run one at a time, waiting for each to complete:

```bash
# Backup
sudo systemctl start resticprofile-backup@profile-vm.service
sudo systemctl status resticprofile-backup@profile-vm.service -l --no-pager
# Wait for completion, then continue...

# Forget (applies retention policy)
sudo systemctl start resticprofile-forget@profile-vm.service
sudo systemctl status resticprofile-forget@profile-vm.service -l --no-pager
# Wait for completion, then continue...

# Prune (removes unused data)
sudo systemctl start resticprofile-prune@profile-vm.service
sudo systemctl status resticprofile-prune@profile-vm.service -l --no-pager
# Wait for completion, then continue...

# Check (verifies repository integrity)
sudo systemctl start resticprofile-check@profile-vm.service
sudo systemctl status resticprofile-check@profile-vm.service -l --no-pager
```

**System Profile** - Run one at a time, waiting for each to complete:

```bash
# Backup
sudo systemctl start resticprofile-backup@profile-system.service
sudo systemctl status resticprofile-backup@profile-system.service -l --no-pager
# Wait for completion, then continue...

# Forget (applies retention policy)
sudo systemctl start resticprofile-forget@profile-system.service
sudo systemctl status resticprofile-forget@profile-system.service -l --no-pager
# Wait for completion, then continue...

# Prune (removes unused data)
sudo systemctl start resticprofile-prune@profile-system.service
sudo systemctl status resticprofile-prune@profile-system.service -l --no-pager
# Wait for completion, then continue...

# Check (verifies repository integrity)
sudo systemctl start resticprofile-check@profile-system.service
sudo systemctl status resticprofile-check@profile-system.service -l --no-pager
```

### Trigger Jobs Sequentially with Automatic Waiting

For convenience, you can use a script that waits for each job to complete:

```bash
# VM Profile - Sequential execution
sudo systemctl start resticprofile-backup@profile-vm.service && \
  sudo systemctl wait resticprofile-backup@profile-vm.service && \
  sudo systemctl start resticprofile-forget@profile-vm.service && \
  sudo systemctl wait resticprofile-forget@profile-vm.service && \
  sudo systemctl start resticprofile-prune@profile-vm.service && \
  sudo systemctl wait resticprofile-prune@profile-vm.service && \
  sudo systemctl start resticprofile-check@profile-vm.service && \
  sudo systemctl wait resticprofile-check@profile-vm.service && \
  echo "All VM jobs completed"

# System Profile - Sequential execution
sudo systemctl start resticprofile-backup@profile-system.service && \
  sudo systemctl wait resticprofile-backup@profile-system.service && \
  sudo systemctl start resticprofile-forget@profile-system.service && \
  sudo systemctl wait resticprofile-forget@profile-system.service && \
  sudo systemctl start resticprofile-prune@profile-system.service && \
  sudo systemctl wait resticprofile-prune@profile-system.service && \
  sudo systemctl start resticprofile-check@profile-system.service && \
  sudo systemctl wait resticprofile-check@profile-system.service && \
  echo "All system jobs completed"
```

### Monitor Job Status

```bash
# Check status of all resticprofile services
sudo systemctl status 'resticprofile-*@profile-*.service'

# Watch logs in real-time
tail -f /home/serveradmin/pve-backup/logs/vm-backup.log
tail -f /home/serveradmin/pve-backup/logs/system-backup.log

# Check when timers will run next
sudo systemctl list-timers --all | grep restic
```

### View Service Logs

```bash
# View logs for a specific job
sudo journalctl -u resticprofile-backup@profile-vm.service -f
sudo journalctl -u resticprofile-backup@profile-system.service -f
```

Note: Manually triggered jobs still send healthcheck.io notifications as configured in `profiles.yaml`.
