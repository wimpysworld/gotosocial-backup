<div align="center">
<img src="https://raw.githubusercontent.com/superseriousbusiness/gotosocial/refs/heads/main/web/assets/logo.png" alt="GotoSocial Backup" width="256" />

# GoToSocial Backup

**A comprehensive backup script for [GoToSocial](https://gotosocial.org/) instances**

**Made with 💝 for 🦥**
</div>

# GoToSocial Backup

A comprehensive backup script for [GoToSocial](https://gotosocial.org/) instances that handles data export, SQLite database backup, and local media file backup with incremental storage using hard-links and optional encryption of sensitive data.

## Features

- Complete instance backup including:
  - GoToSocial data export 🦥
  - SQLite database backup with integrity checking ️🗃️
  - Local media backup of attachments 📎 and custom emojis 🙂
- Incremental backups using hard-links to save space 🤏
- Backup structure that preserves directory hierarchy 📂
- Configurable retention period 📅
- Automatic log rotation ️♻️
- Support for both NixOS ️❄️ and traditional Linux distributions 🐧
- Optional encryption of sensitive data 🔐
- Optional failure notifications via [ntfy.sh](https://ntfy.sh) ️⚠️
- Suitable for sending backups to untrusted remote storage using [rsync](https://rsync.samba.org/) or [rclone](https://rclone.org/) 🚀

### Error Handling

The script includes comprehensive error checking:
- Verifies root access
- Checks for required tools
- Validates database accessibility
- Performs SQLite integrity checks
- Verifies backup completeness

### Caveats 🚨

- Designed to be run in the root context
- There is no support for PostgreSQL databases
- There is no built-in support for sending backups to remote locations
  - Use `rsync`, `rclone` or similar tools to handle that

## Requirements

- Bash
- curl
- gzip
- SQLite3
- rsync
- OpenSSL

## Installation

1. Download the script:
```bash
curl -O https://raw.githubusercontent.com/wimpysworld/gotosocial-backup/gotosocial-backup.sh
```

2. Make it executable:
```bash
chmod 700 backup-gotosocial.sh
chown root:root backup-gotosocial.sh
```

## Configuration

The script can be configured by copying [gotosocial-backup.conf.example](https://raw.githubusercontent.com/wimpysworld/gotosocial-backup/gotosocial-backup.conf.example) `/etc/gotosocial-backup.conf` and editing the following variables accordingly:

```bash
# Root directory for backups
BACKUP_ROOT="/mnt/data/backup/gotosocial"
# Path to GoToSocial configuration file
GTS_CONFIG="/etc/gotosocial/config.yaml"
# Path to the GoToSocial SQLite database
GTS_DB="/var/lib/gotosocial/database.sqlite"
# Retention period in days
RETENTION_DAYS=28
# Encryption passphrase use to encrypt GtS exports and database
PASSPHRASE="mysupersecretpassphrase"
# Your ntfy instance
NTFY_SERVER="ntfy.sh"
# The topic you want to send gotosocial-backup failure alerts to
NTFY_TOPIC="mygotosocial-alerts"

```

## Distribution-specific Notes

### NixOS

On NixOS, the script automatically detects and uses the `gotosocial-admin` wrapper that is available at `/run/current-system/sw/bin/gotosocial-admin`. No additional configuration is needed.

### Traditional Linux Distributions

On traditional Linux distributions, the script requires the `gotosocial` binary to be in the system PATH and uses the specified configuration file path (`GTS_CONFIG`).

## Usage

### Manual Execution

Run the script as root:

```bash
sudo ./gotosocial-backup.sh
```

## Automated Backups

### Using Cron

Add to root's crontab for automated backups:

```bash
sudo crontab -e
```

Add a line for hourly backups:

```bash
0 * * * * /path/to/gotosocial-backup.sh
```

### Using systemd timer

For systems using systemd, you can create a timer unit to schedule backups.
This provides better logging and monitoring capabilities compared to cron.

1. Create the service unit file at `/etc/systemd/system/gotosocial-backup.service`:

```ini
[Unit]
Description=Backup GotoSocial database and local media

[Service]
ExecStart=/path/to/gotosocial-backup
User=root
```

2. Create the timer unit file at `/etc/systemd/system/gotosocial-backup.timer`:

```ini
[Unit]
Description=Run GotoSocial backup periodically

[Timer]
OnBootSec=5min
OnUnitActiveSec=4h
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

Adjust the backup interval by changing `OnUnitActiveSec` to the desired duration.

3. Reload systemd to recognize new units

```bash
sudo systemctl daemon-reload
```

4. Enable timer to run at boot
```bash
sudo systemctl enable gotosocial-backup.timer
```

5. Start the timer
```bash
sudo systemctl start gotosocial-backup.timer
```

6. Verify the timer is active:
```bash
sudo systemctl status gotosocial-backup.timer
```

7. List all timers
```bash
sudo systemctl list-timers
```

8. Monitor backup execution:
```bash
sudo journalctl -u gotosocial-backup.service
```
9. Follow logs in real-time
```bash
sudo journalctl -u gotosocial-backup.service -f
```

The systemd timer configuration includes:
- Random delay of up to 5 minutes to prevent exact-hour execution
- Persistent timing to catch up on missed backups after system downtime
- Proper logging integration with journald

## Backup Structure

The script creates timestamped backup directories:

```
/mnt/data/backup/gotosocial/
├── 20241113_180102
   │  ├── database.sqlite.gz.enc
   │  ├── export.json.gz.enc
   │  └── mnt
   │     └── data
   │        └── gotosocial
   │           └── storage
   │              ├── 01CCJ5GQF7A2KXVDPQ9Q9X23Q2
   │              │  └── attachment
   │              │     └── original
   │              │        ├── 01JCF13JSSDJ0H8E9CWFFW2GVZ.png
   │              │        └── 01JCF13K0R2HJDZWTPWGRNAPZ2.jpeg
   │              ├── 01H7KYSZYN26CWTKT9JNKSZ2XB
   │              │  ├── attachment
   │              │  │  └── original
   │              │  │     └── 01JCF1V56JDJ3SW03V82PFSQ37.png
   │              │  └── emoji
   │              │     └── original
   │              │        └── 01JCJPE20Z1EYX3RTFBVTMM8JB.png
   │              └── 01NATX9MGDJF8ESH0Q4TD5VAV7
   │                 └── attachment
   │                    └── original
   │                       ├── 01JCF2RK21FM491KRKSFTBFXWD.png
   │                       └── 01JCF2X4KN3032X5ZHRZ94H5MQ.png
   ├── backup.log
   └── latest -> /mnt/data/backup/gotosocial/20241113_180102
```

Each backup includes:
- Compressed SQLite database backup (`database.sqlite.gz`) or encrypted (`database.sqlite.gz.enc`)
- Compressed GoToSocial export (`export.json.gz`) or encrypted (`export.json.gz.enc`)
- Local media files (attachments and emojis) with preserved directory structure
- A `latest` symlink pointing to the most recent successful backup

### Log File

The script maintains a log file at `$BACKUP_ROOT/backup.log`.
The log is automatically rotated to keep the most recent 4096 lines by default.

### Backup Retention

The script maintains backups according to these rules:
1. Keeps all backups from the current day
2. Keeps all backups newer than the retention period
3. Removes backups that are both:
   - Older than the retention period
   - Not created on the current day

## References
- https://docs.gotosocial.org/en/latest/admin/backup_and_restore/
- https://litestream.io/alternatives/cron/
