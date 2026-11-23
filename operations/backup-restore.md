# Backup and Restore

This guide describes how to back up and restore the Wanderers-Quill database
and application data. It is written for the self-hoster running Wanderers-Quill
on their own hardware (Raspberry Pi, VPS, or home server).

> **Scope:** This covers the PostgreSQL database and the application
> configuration. It does not cover the operating system or Docker images, which
> are reproducible from the repository.

---

## Why this matters

Self-hosting means backups are your responsibility. If the Raspberry Pi's SD
card fails, or a power outage corrupts the filesystem, or you accidentally
delete a world, the backup is the only thing between you and data loss.

The strategy has two layers:

1. **Local backup** — fast to create and restore, protects against accidental
   deletion and application errors.
2. **Off-site backup** — protects against hardware failure, theft, and fire.

Both are automated with **borgmatic**, a configuration-driven backup tool that
supports PostgreSQL natively .

---

## What gets backed up

| Data | Location | Method |
| :--- | :--- | :--- |
| PostgreSQL database | Docker volume `postgres_data` | `pg_dump` via borgmatic's PostgreSQL hook  |
| Application config | `~/wanderers-quill/.env` | Included in `source_directories` |
| Uploaded map images (future) | `~/wanderers-quill/uploads/` | Included in `source_directories` |

The PostgreSQL data directory is **never** copied directly while the database
is running. That produces inconsistent backups. Instead, borgmatic streams a
`pg_dump` into the backup archive .

---

## 1. Install borgmatic

On the Raspberry Pi (Raspberry Pi OS, Debian-based):

```bash
sudo apt update
sudo apt install borgmatic
```

Verify the installation:

```bash
borgmatic --version
```

---

## 2. Create the borgmatic configuration

Create `/etc/borgmatic/config.yaml` (or `~/.config/borgmatic/config.yaml` for a
single-user setup):

```yaml
# /etc/borgmatic/config.yaml

location:
    source_directories:
        - /home/pi/wanderers-quill/.env
        - /home/pi/wanderers-quill/uploads

    repositories:
        # Local backup (external USB drive)
        - path: /mnt/backup/wanderers-quill.borg
          label: local

        # Off-site backup (BorgBase, rsync.net, or any Borg-compatible host)
        - path: ssh://youruser@yourhost/./wanderers-quill.borg
          label: offsite

storage:
    encryption_passphrase: "your-strong-passphrase-here"
    # Or use a passphrase file:
    # encryption_passphrase: !include /etc/borgmatic/passphrase.txt

retention:
    keep_daily: 7
    keep_weekly: 4
    keep_monthly: 6

# PostgreSQL backup hook
postgresql_databases:
    - name: wanderers
      hostname: localhost
      port: 5432
      username: wanderers
      password: "your-db-password"
      # For Docker, override the dump command to use the container:
      pg_dump_command: docker exec postgres pg_dump
```

**Notes on the configuration:**

- `source_directories` includes the `.env` file so a restore includes the
  application's secrets. If you prefer to keep secrets out of backups, omit it
  and document the required variables separately.
- `postgresql_databases` uses borgmatic's built-in hook, which streams the dump
  through a named pipe (no temporary files on disk) .
- The `pg_dump_command` override is necessary if PostgreSQL runs in Docker.
  Adjust the container name (`postgres`) to match your `docker-compose.yml`.
- For off-site, any Borg-compatible host works (BorgBase, rsync.net, a friend's
  server, or a second machine you control).

---

## 3. Initialize the repositories

Before the first backup, initialize both Borg repositories:

```bash
borgmatic init --encryption repokey
```

This initializes the local and off-site repositories. Borg will ask for the
encryption passphrase (or read it from the configuration).

**Export the repository key** and store it somewhere safe (not on the Pi):

```bash
borg key export /mnt/backup/wanderers-quill.borg /tmp/wanderers-key.txt
# Copy /tmp/wanderers-key.txt to a password manager or printed copy
```

Without the key and passphrase, the backup is unreadable. This is by design.

---

## 4. Test the backup manually

Run a backup now:

```bash
borgmatic create --verbosity 1 --list
```

Check that it succeeded:

```bash
borgmatic list --last 3
```

You should see an archive with today's date. Verify the database dump is
included:

```bash
borgmatic extract --archive latest --dry-run --list
```

Look for a path containing `postgresql_databases/wanderers`.

---

## 5. Schedule automated backups

Use a systemd timer (preferred on Raspberry Pi OS) or cron.

### Option A: systemd timer (recommended)

Create `/etc/systemd/system/borgmatic.timer`:

```ini
[Unit]
Description=Daily borgmatic backup

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

Enable and start:

```bash
sudo systemctl enable borgmatic.timer
sudo systemctl start borgmatic.timer
```

### Option B: cron

Add to root's crontab:

```
0 3 * * * /usr/bin/borgmatic create --verbosity 1
```

This runs daily at 03:00.

---

## 6. Restore

### Restore the database

If the database is corrupted or lost, restore the most recent dump:

```bash
borgmatic restore --archive latest --data-source wanderers
```

By default, borgmatic restores to the database configured in the
`postgresql_databases` section. To restore to a different hostname or database,
use `--hostname`, `--port`, `--database`, and related flags .

**Prerequisite:** The target database must exist before restoring. Create it if
needed:

```bash
docker exec postgres createdb -U wanderers wanderers_restore
```

Then restore into it:

```bash
borgmatic restore --archive latest --data-source wanderers --database wanderers_restore
```

### Restore application files

If you need to restore the `.env` file or uploaded images:

```bash
borgmatic extract --archive latest --destination /tmp/restore
```

Then copy the files back to their original locations.

### Full disaster recovery (new hardware)

1. Install Raspberry Pi OS and Docker.
2. Install borgmatic.
3. Restore the borgmatic configuration from your password manager or a printed
   copy.
4. Import the Borg repository key:

   ```bash
   borg key import /mnt/backup/wanderers-quill.borg /path/to/key.txt
   ```

5. Restore the database and files as described above.
6. Start the application with `docker compose up -d`.

---

## 7. Verify backups regularly

A backup you never restore is a backup you don't know works. Once a month:

1. List the archives: `borgmatic list --last 5`
2. Extract a file from the latest archive to `/tmp` and check it opens.
3. Once a quarter, do a full restore into a test database and confirm the data
   is intact.

Per invariant I-18, every bug fix includes a test. Treat backups the same way:
every backup strategy includes a restore test.

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
| :--- | :--- | :--- |
| `permission denied` on `/mnt/backup` | USB drive not mounted or wrong permissions | Check `mount` output; ensure the drive is mounted and writable by root |
| `pg_dump: command not found` | Docker container name mismatch | Update `pg_dump_command` in the config to match your container name |
| `Failed to create lock` | Another borgmatic process is running | Check for stale locks: `borg break-lock /mnt/backup/wanderers-quill.borg` |
| Off-site backup fails | SSH key or network issue | Test `ssh youruser@yourhost` manually; check BorgBase/rsync.net status |

---

## 9. What this does not cover

- **Encryption key management beyond borgmatic.** If you lose the passphrase
  and the exported key, the backup is unrecoverable.
- **Docker volume backups.** The database is backed up via `pg_dump`, not by
  copying the Docker volume. This is intentional: a `pg_dump` is consistent and
  portable across PostgreSQL versions.
- **Application-level exports.** A future feature may allow exporting a world
  as JSON. That is not part of this guide.
