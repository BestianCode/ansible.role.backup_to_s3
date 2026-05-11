# ng-backup

`ng-backup` provisions three host-side helpers:

- `ng-backup.sh` (scheduled via cron) stops services, archives the requested directories, ships them to S3, and rotates local copies.
- `ng-restore.sh` provides a quick way to see which timestamps are available in the S3 prefix and download one of them back onto the host.
- `ng-purge.sh` (scheduled via cron) applies a graduated retention policy to S3 backups, thinning older snapshots automatically.

Because all three scripts embed AWS credentials directly, restrict filesystem permissions to the account that owns them (root by default).

## Variables worth overriding

| Variable | Default | Purpose |
| --- | --- | --- |
| `aws_s3_backup_bucket` | `backups-seven-tips` | Destination S3 bucket. |
| `aws_s3_backup_bucket_folder` | `gitlab` | Prefix within the bucket where dated directories are stored. |
| `aws_s3_backup_folders` | `/git`, `/opt/gitlab`, `/var/opt/gitlab` | Directories that are archived each run. |
| `aws_s3_backup_local_folder` | `/git-backups/s3` | Local root for timestamped directories. |
| `aws_s3_backup_local_retention` | `7` | How many dated folders remain on disk. |
| `aws_s3_backup_stop_services` | `['gitlab-runsvdir.service','gitlab-runner.service','nginx.service']` | Services stopped before archiving (empty list disables). |
| `aws_s3_backup_pre_stop_commands` | `[]` | Arbitrary commands invoked before the service stop loop (e.g., `gitlab-ctl stop`). |
| `aws_s3_backup_pre_stop_strict` | `false` | When `true`, any pre-stop command failure aborts the backup; when `false`, failures are logged but the run continues. |
| `aws_s3_backup_post_start_commands` | `[]` | Commands executed after services start again. |
| `aws_s3_backup_reboot_after` | `false` | When `true`, the host schedules a reboot after a successful backup. |
| `aws_s3_backup_script_path` | `/root/sbin/ng-backup.sh` | Where the backup script lands. |
| `aws_s3_backup_restore_script_path` | `/root/sbin/ng-restore.sh` | Where the restore helper lands. |
| `aws_s3_backup_restore_folder` | `/git-backups/s3/restore` | Target directory for downloaded backups. |
| `aws_s3_backup_exec_path` | `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/gitlab/bin` | PATH injected into both scripts so cron can find `gitlab-ctl`, `aws`, etc. |
| `aws_s3_backup_schedule` | `0 1 * * *` | Cron expression used to run `ng-backup.sh`. |
| `aws_s3_backup_archive_format` | `tar.gz` | Archive format (`tar.gz` or `7z`). |
| `aws_s3_backup_encryption_enabled` | `false` | Encrypt each produced artifact before upload. |
| `aws_s3_backup_encryption_cipher` | `aes-256-cbc` | OpenSSL cipher used with `openssl enc` when encryption is enabled. |
| `aws_s3_backup_encryption_passphrase` | `""` | Passphrase for encryption (store in Ansible Vault). |

To use password-protected 7z archives instead of gzip, set:

```yaml
aws_s3_backup_archive_format: "7z"
aws_s3_backup_encryption_passphrase: "vault-or-secret-value"
aws_s3_backup_encryption_enabled: false
```

In this mode the script uses `7z a -t7z -mhe=on -p...` to produce encrypted `.7z` artifacts directly.

### S3 purge variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `aws_s3_backup_purge_enabled` | `false` | Master toggle for the purge script and its cron job. |
| `aws_s3_backup_purge_script_path` | `/root/sbin/ng-purge.sh` | Where the purge script lands. |
| `aws_s3_backup_purge_log_file` | `/var/log/ng-purge.log` | Log file for purge runs. |
| `aws_s3_backup_purge_schedule` | `30 3 * * *` | Cron expression for the purge job (runs after backups complete). |
| `aws_s3_backup_purge_dry_run` | `true` | When `true`, the script logs what it **would** delete but makes no changes. **Set to `false` after verifying the output.** |
| `aws_s3_backup_purge_keep_all_days` | `7` | Level 1 — keep every backup made within the last N days. |
| `aws_s3_backup_purge_keep_daily_days` | `23` | Level 2 — keep 1 backup per day for the next N days. |
| `aws_s3_backup_purge_keep_weekly_days` | `60` | Level 3 — keep 1 backup per week for the next N days. |
| `aws_s3_backup_purge_keep_monthly_days` | `365` | Level 4 — keep 1 backup per month for the next N days. |
| `aws_s3_backup_purge_delete_after_days` | `0` | Level 5 — delete everything older. **`0` disables this level**, so monthly backups are kept forever. |

All five windows are contiguous and stack on top of each other:

```text
today ──────────────────────────────────────────────────────────────► past
│  keep all  │  1 per day  │  1 per week  │  1 per month  │  delete  │
│   7 days   │   23 days   │   60 days    │   365 days    │  (rest)  │
```

Setting any level to `0` collapses that window — backups simply fall through to the next level.  For example, with `keep_all=7, keep_daily=0, keep_weekly=60`, there is no daily tier; the weekly tier starts immediately after the keep-all window.

See `defaults/main.yml` for the full catalog, including AWS credentials, logging, and compression controls.

## Using the restore helper

1. Run `ng-restore.sh` with no arguments to list the available timestamps. Each entry is numbered for convenience.
2. Run `ng-restore.sh <number>` to download the selected snapshot into `aws_s3_backup_restore_folder/<timestamp>`.
3. Extract whichever tarball you need from the downloaded directory.

The script simply wraps `aws s3 sync`, so you can clean up restored directories at any time without affecting the data stored in S3.

If encryption is enabled, downloaded artifacts have `.enc` suffix and must be decrypted before extraction. Example:

```bash
openssl enc -d -aes-256-cbc -pbkdf2 \
  -pass env:BACKUP_ENCRYPTION_PASSPHRASE \
  -in archive.tar.gz.enc \
  -out archive.tar.gz
```

## Using the purge script

The purge script is designed to be safe by default:

1. Enable it by setting `aws_s3_backup_purge_enabled: true`.
2. Leave `aws_s3_backup_purge_dry_run: true` (the default) and let it run for a few days.  Check `/var/log/ng-purge.log` to verify the retention decisions look correct.
3. Set `aws_s3_backup_purge_dry_run: false` to begin actual deletions.

You can also run the script manually at any time to inspect or test: `sudo /root/sbin/ng-purge.sh`.
