---
name: backint
description: >
  SAP HANA BACKINT-to-GCS backup skill. Use for: setting up, configuring, and
  troubleshooting HANA backups to Google Cloud Storage via the Google Cloud Agent
  for SAP (google_cloud_sap_agent backint) on sapidess4 (S/4HANA, SID S4H,
  tenants FIV + PIT) and on bw4hana (BW/4HANA, SID BWH). Covers parameters.json,
  hdbbackint wrapper, global.ini [backup] config, systemd service, GCS bucket
  sap-hana-backint layout, catalog/log backups, replication of config to a new
  tenant or system. Also covers the parallel two-phase SQL Server → GCS backup
  flow for SAP on SQL Server (SQ1 on sap-sql-ides) — same bucket under
  sql_server_backup/ prefix, using gsutil (not hdbbackint).
user_invokable: true
metadata:
  team: data
---

# SAP HANA BACKINT → GCS — Expert Context

End-to-end reference for HANA data/log/catalog backups to the shared GCS bucket
`sap-hana-backint` via Google Cloud Agent for SAP. Covers both systems currently
wired up:

- **sapidess4** — SID `S4H`, tenants `FIV` (primary) + `PIT` (HANA Cockpit DBA)
- **bw4hana** — SID `BWH` (single tenant)

---

## Architecture

HANA invokes `hdbbackint` whenever a backup is triggered. `hdbbackint` is a
symlink to a one-line wrapper that forwards to the Google Cloud Agent:

```
hdbsql  →  BACKUP DATA USING BACKINT ('...')
HANA    →  /usr/sap/<SID>/SYS/global/hdb/opt/hdbbackint
           └─ symlink → /usr/sap/<SID>/SYS/global/hdb/opt/backint/backint-gcs/backint
              └─ `#!/bin/bash` wrapper: `/usr/bin/google_cloud_sap_agent backint "$@"`
                 └─ reads parameters.json (sibling file)
                    └─ uploads to gs://sap-hana-backint/<TENANT>/<hana-path>/<timestamp>.bak
```

Parameters.json tells the agent which GCS bucket to use, which service-account
key to authenticate with, and whether to compress + log to cloud. HANA's
global.ini `[backup]` section tells HANA to use this parameters.json for data,
catalog, and log backups. HANA's `[persistence]` section gives a local fallback
base path (used when backint is bypassed or for catalog staging).

Runtime component:
- **`google-cloud-sap-agent.service`** (systemd) — long-running agent providing
  monitoring + the `backint` subcommand. Must be `enabled --now` on every host
  that does backint backups. Binary: `/usr/bin/google_cloud_sap_agent`.
- Service config: `/etc/google-cloud-sap-agent/configuration.json`
- Agent logs: `/var/log/google-cloud-sap-agent/*.log`

---

## GCS Bucket Layout

Bucket: `gs://sap-hana-backint/` (shared across all HANA systems, tenant-prefixed)

| Prefix | System | Tenant | Notes |
|--------|--------|--------|-------|
| `FIV/` | sapidess4 | FIV | Primary S/4HANA backups — weekly full Saturday 09:33 LA |
| `PIT/` | sapidess4 | PIT (SYSTEMDB on port 39613) | HANA Cockpit DBA tenant |
| `BWH/` | bw4hana | BWH (+ SYSTEMDB) | BW/4HANA backups (added 2026-04-16) |
| `S4HANA_SAP_BACKUP/` | legacy/manual | — | Ad-hoc |
| `SAP_ON_ORACLE_BACKUP/` | sapidesecc8 | — | brbackup dumps (separate flow) |

Full HANA backup file key format: `<TENANT>/usr/sap/<SID>/SYS/global/hdb/backint/<TENANT>/<filename>/<epochMs>.bak`

All `.bak` files are gzip-compressed by the agent (`"compress": true` in
parameters.json). Log backups are tiny (often 8–12 KB compressed) and arrive
every minute; data backups are multi-GB.

---

## Per-system configuration

### sapidess4 — SID S4H (FIV + PIT)

| File | Value |
|------|-------|
| global.ini (custom) | `/usr/sap/FIV/SYS/global/hdb/custom/config/global.ini` |
| parameters.json | `/usr/sap/FIV/SYS/global/hdb/opt/backint/backint-gcs/parameters.json` |
| backint wrapper | `/usr/sap/FIV/SYS/global/hdb/opt/backint/backint-gcs/backint` |
| hdbbackint symlink | `/usr/sap/FIV/SYS/global/hdb/opt/hdbbackint` |
| SA key | `/usr/sap/FIV/home/internal-sales-4b50698e74ec.json` |
| basepath_{data,log,catalog}backup | `/SUMHANA/backup` (NFS from saphvrhub) |
| Parallel channels | 4 |
| Tenants | FIV (port 30015, SAPHANADB) + PIT (port 39613, SYSTEM) |
| Weekly full backup | Saturdays 09:33 LA (America/Los_Angeles, server TZ) |

**parameters.json**:
```json
{
  "bucket": "sap-hana-backint",
  "compress": true,
  "service_account_key": "/usr/sap/FIV/home/internal-sales-4b50698e74ec.json",
  "log_to_cloud": true
}
```

### bw4hana — SID BWH

| File | Value |
|------|-------|
| global.ini (custom) | `/usr/sap/BWH/SYS/global/hdb/custom/config/global.ini` |
| parameters.json | `/usr/sap/BWH/SYS/global/hdb/opt/backint/backint-gcs/parameters.json` |
| backint wrapper | `/usr/sap/BWH/SYS/global/hdb/opt/backint/backint-gcs/backint` |
| hdbbackint symlink | `/usr/sap/BWH/SYS/global/hdb/opt/hdbbackint` |
| SA key | `/usr/sap/BWH/home/internal-sales-4b50698e74ec.json` |
| basepath_{data,log,catalog}backup | `/SUMHANA/BWH_backup` (SID-specific subdir on NFS) |
| Parallel channels | 4 |
| Tenants | BWH (port 30015, SYSTEM/SAPHANADB) + SYSTEMDB (port 30013, SYSTEM) |
| HANA server TZ | UTC |
| Configured | 2026-04-16 (mirror of sapidess4 config) |

**global.ini [backup] + [persistence] appended sections:**
```ini
[persistence]
basepath_datavolumes = /hana/data/BWH
basepath_logvolumes = /hana/log/BWH
basepath_databackup = /SUMHANA/BWH_backup
basepath_logbackup = /SUMHANA/BWH_backup
basepath_catalogbackup = /SUMHANA/BWH_backup

[backup]
data_backup_parameter_file = /usr/sap/BWH/SYS/global/hdb/opt/backint/backint-gcs/parameters.json
parallel_data_backup_backint_channels = 4
catalog_backup_parameter_file = /usr/sap/BWH/SYS/global/hdb/opt/backint/backint-gcs/parameters.json
log_backup_parameter_file = /usr/sap/BWH/SYS/global/hdb/opt/backint/backint-gcs/parameters.json
log_backup_using_backint = true
catalog_backup_using_backint = true
```

---

## Replicating BACKINT to a new HANA system

Substitute `<SID>` with the tenant SID (uppercase), `<HOST>` with the host, and
`<ADMUSER>` with the SID's OS admin (e.g. `bwhadm`, `fivadm`).

**1. Install Google Cloud Agent for SAP** (idempotent — usually present on
SAP-on-GCE images):
```bash
which google_cloud_sap_agent || { echo "install the agent from GCP Marketplace"; exit 1; }
systemctl enable --now google-cloud-sap-agent
```

**2. Copy the shared service-account key** from a working host (idempotent —
same key is fine for multiple systems as long as the SA has `storage.objectAdmin`
on the bucket):
```bash
ssh root@sapidess4 "cat /usr/sap/FIV/home/internal-sales-4b50698e74ec.json" \
  | ssh root@<HOST> "cat > /usr/sap/<SID>/home/internal-sales-4b50698e74ec.json \
                     && chown <ADMUSER>:sapsys /usr/sap/<SID>/home/internal-sales-4b50698e74ec.json \
                     && chmod 600 /usr/sap/<SID>/home/internal-sales-4b50698e74ec.json"
```

**3. Create backint dir + wrapper + parameters.json + symlink** (on target):
```bash
mkdir -p /usr/sap/<SID>/SYS/global/hdb/opt/backint/backint-gcs
cat > /usr/sap/<SID>/SYS/global/hdb/opt/backint/backint-gcs/backint <<'BKE'
#!/bin/bash
/usr/bin/google_cloud_sap_agent backint "$@"
exit $?
BKE
cat > /usr/sap/<SID>/SYS/global/hdb/opt/backint/backint-gcs/parameters.json <<PJSON
{
  "bucket": "sap-hana-backint",
  "compress": true,
  "service_account_key": "/usr/sap/<SID>/home/internal-sales-4b50698e74ec.json",
  "log_to_cloud": true
}
PJSON
chmod +x /usr/sap/<SID>/SYS/global/hdb/opt/backint/backint-gcs/backint
ln -sfn /var/log/google-cloud-sap-agent /usr/sap/<SID>/SYS/global/hdb/opt/backint/backint-gcs/logs
chown -R <ADMUSER>:sapsys /usr/sap/<SID>/SYS/global/hdb/opt/backint
ln -sfn /usr/sap/<SID>/SYS/global/hdb/opt/backint/backint-gcs/backint /usr/sap/<SID>/SYS/global/hdb/opt/hdbbackint
chown -h <ADMUSER>:sapsys /usr/sap/<SID>/SYS/global/hdb/opt/hdbbackint
```

**4. Create local basepath** (SID-specific subdir on shared NFS to avoid tenant
collisions):
```bash
su - <ADMUSER> -c "mkdir -p /SUMHANA/<SID>_backup"
```

**5. Update global.ini online via SQL** (no HANA restart needed):
```sql
-- Run as SYSTEM on SYSTEMDB
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM') SET
  ('backup', 'data_backup_parameter_file') = '/usr/sap/<SID>/SYS/global/hdb/opt/backint/backint-gcs/parameters.json' WITH RECONFIGURE;
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM') SET
  ('backup', 'catalog_backup_parameter_file') = '/usr/sap/<SID>/SYS/global/hdb/opt/backint/backint-gcs/parameters.json' WITH RECONFIGURE;
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM') SET
  ('backup', 'log_backup_parameter_file') = '/usr/sap/<SID>/SYS/global/hdb/opt/backint/backint-gcs/parameters.json' WITH RECONFIGURE;
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM') SET
  ('backup', 'parallel_data_backup_backint_channels') = '4' WITH RECONFIGURE;
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM') SET
  ('backup', 'log_backup_using_backint') = 'true' WITH RECONFIGURE;
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM') SET
  ('backup', 'catalog_backup_using_backint') = 'true' WITH RECONFIGURE;
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM') SET
  ('persistence', 'basepath_databackup') = '/SUMHANA/<SID>_backup' WITH RECONFIGURE;
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM') SET
  ('persistence', 'basepath_logbackup') = '/SUMHANA/<SID>_backup' WITH RECONFIGURE;
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM') SET
  ('persistence', 'basepath_catalogbackup') = '/SUMHANA/<SID>_backup' WITH RECONFIGURE;
```

**6. Verify with a test backup** (SYSTEMDB is fast, use it first):
```bash
/usr/sap/<SID>/hdbclient/hdbsql -n localhost -i 00 -d SYSTEMDB -u SYSTEM -p '<pwd>' \
  -x -a "BACKUP DATA FOR SYSTEMDB USING BACKINT ('initial_test_$(date +%Y%m%d_%H%M%S)')"
# RC=0 and appear in gs://sap-hana-backint/<SID>/
gsutil ls gs://sap-hana-backint/<SID>/ | head
tail -15 /var/log/google-cloud-sap-agent/*.log
```

Full tenant data backup (larger, run when you've got time):
```sql
BACKUP DATA FOR <TENANT> USING BACKINT ('FULL_DATA_BACKUP_<TENANT>_initial')
```

---

## Verification SQL

### Last successful data backup
```sql
SELECT TOP 1 SYS_END_TIME
FROM SYS.M_BACKUP_CATALOG
WHERE STATE_NAME = 'successful'
  AND ENTRY_TYPE_NAME = 'complete data backup'
ORDER BY SYS_END_TIME DESC;
```

### Backup catalog summary
```sql
SELECT ENTRY_TYPE_NAME, STATE_NAME, COUNT(*)
FROM SYS.M_BACKUP_CATALOG
GROUP BY ENTRY_TYPE_NAME, STATE_NAME
ORDER BY 1, 2;
```

### Backups currently in progress
```sql
SELECT STATE_NAME, SECONDS_BETWEEN(SYS_START_TIME, CURRENT_TIMESTAMP) AS ELAPSED_SEC
FROM SYS.M_BACKUP_PROGRESS
WHERE ENTRY_TYPE_NAME = 'complete data backup'
ORDER BY SYS_START_TIME DESC;
```

### HANA license expiry (shown in the BW/4HANA cockpit tile)
```sql
SELECT SYSTEM_ID, EXPIRATION_DATE, PERMANENT, VALID, PRODUCT_NAME FROM M_LICENSE;
```

---

## Agent CLI Commands — Install, Configure, Validate, Diagnose

The Google Cloud Agent for SAP ships with helper commands that avoid manual
symlink/wrapper/parameters.json work. Use these instead of the manual replication
procedure above for new systems.

### `installbackint` — Install agent backint files

Deploys the wrapper, `hdbbackint` symlink, and a starter `parameters.json` to
`/usr/sap/<SID>/SYS/global/hdb/opt/backint/backint-gcs/`.

```bash
# Install using the current $SAPSYSTEMNAME env var
/usr/bin/google_cloud_sap_agent installbackint

# Or specify SID explicitly (lowercase)
/usr/bin/google_cloud_sap_agent installbackint -sid=bwh
```

If a legacy Cloud Storage Backint agent is present, `installbackint` disables it
and moves its files to a recoverable directory.

### `configurebackint` — Edit parameters.json safely

Avoid hand-editing parameters.json. Use this instead:

```bash
/usr/bin/google_cloud_sap_agent configurebackint \
  -f="/usr/sap/BWH/SYS/global/hdb/opt/backint/backint-gcs/parameters.json" \
  -bucket="sap-hana-backint"

# For on-premises / non-Compute-Engine hosts (adds SA key + disables cloud logging/metrics)
/usr/bin/google_cloud_sap_agent configurebackint \
  -f="/path/parameters.json" \
  -bucket="sap-hana-backint" \
  -service_account_key="/path/key.json" \
  -log_to_cloud=false \
  -send_metrics_to_monitoring=false
```

### `status -f=backint` — Validate config + IAM (v3.7+)

Returns IAM permissions status, configuration sources (file vs default), and all
effective parameter values. Run this first when troubleshooting.

```bash
sudo /usr/bin/google_cloud_sap_agent status \
  -b="/usr/sap/BWH/SYS/global/hdb/opt/backint/backint-gcs/parameters.json" \
  -f="backint"
```

### `backint -f=diagnose` — Self-test backup/restore

Runs a full backup-upload-download-restore cycle end-to-end against GCS using
the configured parameters. Validates the entire pipeline without involving
HANA. Requires **≥18 GB free disk** (2–3 GB beyond `diagnose_file_max_size_gb`
for temp files).

```bash
sudo /usr/bin/google_cloud_sap_agent backint \
  -u=BWH \
  -p="/usr/sap/BWH/SYS/global/hdb/opt/backint/backint-gcs/parameters.json" \
  -f=diagnose

# Override defaults if /tmp is constrained
sudo /usr/bin/google_cloud_sap_agent backint -u=BWH \
  -p=/path/parameters.json -f=diagnose \
  -diagnose_file_max_size_gb=4 \
  -diagnose_tmp_directory=/SUMHANA/diagnose_tmp
```

---

## parameters.json — Full Reference

Our current configs only use 4 parameters. The agent supports many more for
performance tuning, encryption, retention, and system-copy workflows. Full table:

### Core

| Parameter | Default | Notes |
|-----------|---------|-------|
| `bucket` | (required) | Target GCS bucket. Ours: `sap-hana-backint` |
| `recovery_bucket` | — | Separate bucket for restore. Incompatible with `CHECK ACCESS USING BACKINT`. Handy for system-copy/refresh (v3.1+) |
| `folder_prefix` | — | Prefix inside the bucket (`folder1/folder2`). Changes object paths — test first |
| `recovery_folder_prefix` | — | Prefix for restore operations (v3.1+) |
| `shorten_folder_path` | false | Shortens object path lengths (v3.3+) |
| `service_account_key` | — | Required only off-Compute-Engine (BMS/on-prem). Full path to JSON key |

### Performance tuning

| Parameter | Default | Notes |
|-----------|---------|-------|
| `parallel_streams` | 1 | 1–32. For data backups, prefer HANA's `parallel_data_backup_backint_channels` instead. For log/catalog, combine with `xml_multipart_upload=true` |
| `parallel_recovery_streams` | 1 | 1–32 (v3.7+). **Not** with compressed backups |
| `xml_multipart_upload` | false | Enables XML API multipart uploads. Required if `parallel_streams>1`. Auto-sets `parallel_streams=16` if unspecified (v3.2+) |
| `buffer_size_mb` | 100 | Up to 250. Larger buffer = more throughput + more memory + more data to resend on retry. Memory = `buffer_size × parallel_streams` |
| `rate_limit_mb` | unlimited | Cap outbound bandwidth during backup/restore |
| `threads` | nproc | Worker threads. Change only per Customer Care guidance |
| `retries` | 5 | GCS read/write retry count |
| `retry_backoff_initial` | 10 s | Initial exponential-backoff interval |
| `retry_backoff_max` | 300 s | Max backoff |
| `retry_backoff_multiplier` | 2.0 | Must be > 1.0 |

### Compression, logging & metrics

| Parameter | Default | Notes |
|-----------|---------|-------|
| `compress` | false | Enables gzip. **Google's recommendation: don't use it** — higher CPU, lower throughput. Only enable if storage cost matters. (We currently have `true`.) |
| `log_to_cloud` | true | Routes backint logs to Cloud Logging. Set `false` on-prem |
| `log_level` | INFO | DEBUG / INFO / WARNING / ERROR. Change only per Customer Care |
| `log_delay_sec` | 60 | Progress-log interval |
| `send_metrics_to_monitoring` | true | Cloud Monitoring metrics. Set `false` on-prem (v3.3+) |

### Storage / metadata

| Parameter | Default | Notes |
|-----------|---------|-------|
| `storage_class` | bucket default (v3.7+) / STANDARD earlier | STANDARD, NEARLINE, COLDLINE, ARCHIVE (v3.2+) |
| `metadata` | `{"X-Backup-Type": "PIPE"/"FILE"}` | Custom KV map. Don't override `X-Backup-Type` (v3.3+) |
| `custom_time` | — | RFC 3339 or `UTCNow+Nd`. Sets object `Custom-Time` metadata (v3.4+, future dates v3.6+) |
| `client_endpoint` | storage.googleapis.com | Rarely modified |

### Encryption

| Parameter | Default | Notes |
|-----------|---------|-------|
| `encryption_key` | — | Path to base64 AES-256 CSEK. Mutually exclusive with `kms_key`, `parallel_streams` |
| `kms_key` | — | `projects/P/locations/L/keyRings/R/cryptoKeys/K`. Mutually exclusive with `encryption_key`, `parallel_streams` |

### Retention (v3.7+)

| Parameter | Default | Notes |
|-----------|---------|-------|
| `object_retention_time` | — | RFC 3339 or `UTCNow+Nd`. Requires `object_retention_mode` |
| `object_retention_mode` | — | `Locked` or `Unlocked`. Requires `object_retention_time` |

### Diagnostics

| Parameter | Default | Notes |
|-----------|---------|-------|
| `file_read_timeout_ms` | 60000 | Timeout opening backup file |
| `diagnose_file_max_size_gb` | 16 | Max file size for `-f=diagnose` (v3.3+) |
| `diagnose_tmp_directory` | /tmp/backint-diagnose | Temp dir for diagnose (v3.3+) |

### Constraint matrix

- Parallel uploads **incompatible with**: retention policies on bucket, `encryption_key`, `kms_key` → agent exits with status 1
- `parallel_recovery_streams` **incompatible with** `compress=true` backups
- `recovery_bucket` **incompatible with** `CHECK ACCESS USING BACKINT` SQL clause

---

## Multiple Parameter Files (Recommended for Production)

HANA lets you point data / log / catalog backups at **separate** parameters.json
files. This is the standard pattern for tuning each backup type differently.

Recommended split:

| Backup type | File | Typical tuning |
|-------------|------|----------------|
| Data | `parameters_data.json` | HANA's `parallel_data_backup_backint_channels=8-16` (global.ini), no `parallel_streams` here |
| Log | `parameters_log.json` | `parallel_streams=8`, `xml_multipart_upload=true`, smaller `buffer_size_mb` |
| Catalog | `parameters_catalog.json` | Same as log |

Point HANA at each file via `global.ini`:

```ini
[backup]
data_backup_parameter_file = /usr/sap/BWH/.../parameters_data.json
log_backup_parameter_file = /usr/sap/BWH/.../parameters_log.json
catalog_backup_parameter_file = /usr/sap/BWH/.../parameters_catalog.json
```

We currently use **one** shared parameters.json for all three on both systems.
Works fine for this scale; split when performance matters.

---

## Performance & Operational Notes

- **Compression tradeoff:** Google recommends leaving `compress=false`. Our two
  systems currently use `compress=true` — deliberate trade for lower GCS cost
  on this IDES/demo landscape. Not recommended for prod workloads where
  throughput matters.
- **Parallel data backup channels:** set
  `parallel_data_backup_backint_channels=8` (or 16) in global.ini `[backup]`
  *instead of* `parallel_streams` in parameters.json for data backups. Both
  systems currently set `4` — can be increased if backup windows shrink.
- **XML multipart uploads** (v3.2+) for log/catalog: small frequent files
  benefit most. Enable in a dedicated `parameters_log.json`, not the shared
  file.
- **Lifecycle rule on bucket** — Google recommends an
  `AbortIncompleteMultipartUpload` lifecycle rule (7 days) to clean up after
  failed uploads.
- **Database isolation feature:** If HANA is deployed with database isolation,
  extra permissions/config are required — not applicable to either of our
  systems.
- **Scale-out deployments:** Agent must be installed on every node.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `createDirectory(path='/usr/sap/<SID>/HDB00/backup/log/DB_<TENANT>/'): Permission denied (rc=13)` in `/usr/sap/<SID>/HDB00/<host>/trace/DB_<TENANT>/backup.log` | `/usr/sap/<SID>/HDB00/backup` owned by `root:root` (created by `hdbnsutil -initTopology`, never chowned) | `chown -R <ADMUSER>:sapsys /usr/sap/<SID>/HDB00/backup` |
| `ZoneInfoNotFoundError 'No time zone found with key America/Los_Angeles'` in server.py backup query | `tzdata` package missing from miniconda | `/usr/sap/miniconda/bin/python3 -m pip install tzdata` (on sapidesecc8). Always use `-m pip` — the `pip` shebang is broken on sapidesecc8. |
| `-10709: Connection failed (RTE:[89006] System call 'connect' failed, rc=111:Connection refused (localhost:30015))` | Tenant is down or mid-recovery. SYSTEMDB (30013) often stays up while tenant's indexserver (30015) is still binding. | Wait for `sapcontrol -nr 00 -function GetProcessList` to show `hdbindexserver-<TENANT>, GREEN, Running`. Tail `/usr/sap/<SID>/HDB00/<host>/trace/DB_<TENANT>/backup.log` to watch recovery progress. |
| `BACKUP … USING BACKINT` fails with `opening /usr/sap/<SID>/SYS/global/hdb/opt/hdbbackint` | hdbbackint symlink missing or points to wrong path | Recreate symlink (step 3 above) |
| Agent logs `failed to authenticate with storage` | SA key file missing, wrong permissions (must be readable by `<ADMUSER>`), or the SA lost `storage.objectAdmin` on the bucket | `ls -la /usr/sap/<SID>/home/internal-sales-*.json` → fix ownership/mode; verify bucket IAM |
| Agent logs `bucket not found` | Wrong bucket name in parameters.json | Must be exactly `sap-hana-backint` |
| Log backups never upload but data backup works | `log_backup_using_backint = false` or `log_backup_parameter_file` unset | Set both in global.ini [backup] via ALTER SYSTEM |
| Cockpit "Last backup" shows empty on S/4HANA FIV tile but PIT works | `tzdata` missing in miniconda → FIV code path uses ZoneInfo while PIT doesn't | Install tzdata as above |
| SAPLIKEY query errors with `invalid schema name: SCHEMA_NAME` | hdbsql `-quiet -C` leaves column name in output; script reads header as data | Use `-x -a` (no `-quiet -C`) so hdbsql omits header |
| Unsure if config / IAM is correct before triggering first backup | — | Run `google_cloud_sap_agent status -b=/path/parameters.json -f=backint` — returns IAM and config validation (v3.7+) |
| Want to test GCS upload/download before HANA backup | — | Run `google_cloud_sap_agent backint -u=<SID> -p=/path/parameters.json -f=diagnose` (needs 18 GB free) |
| Agent exits with status 1 immediately | Incompatible option combination (e.g. `parallel_streams` + `kms_key`, or parallel + retention policy) | Check parameters.json against the constraint matrix in parameters.json reference |

---

## File/path cheat sheet

```
/usr/sap/<SID>/SYS/global/hdb/opt/
├── hdbbackint                    → backint-gcs/backint  (symlink, HANA hook)
└── backint/
    └── backint-gcs/
        ├── backint               (wrapper: exec google_cloud_sap_agent backint "$@")
        ├── parameters.json       (bucket, SA key, compress, log_to_cloud)
        └── logs                  → /var/log/google-cloud-sap-agent  (symlink)

/usr/sap/<SID>/home/internal-sales-4b50698e74ec.json   (GCS SA key, 600 perms, <ADMUSER>)
/usr/sap/<SID>/SYS/global/hdb/custom/config/global.ini  (custom HANA config)
/usr/sap/<SID>/HDB00/<host>/trace/DB_<TENANT>/backup.log  (tenant backup/recovery log)
/var/log/google-cloud-sap-agent/*.log                    (agent runtime logs)
/SUMHANA/<SID>_backup/                                   (local basepath fallback, NFS)
/etc/google-cloud-sap-agent/configuration.json            (agent systemd config)
```

---

## Cockpit integration

Both cockpits show the last successful HANA data backup from `SYS.M_BACKUP_CATALOG`:

| Cockpit page | Endpoint | HANA tile element |
|--------------|----------|-------------------|
| `SAP_S4HANA_2023.html` | `GET /sap_skills/api/system_status` | `hana-backup-date` (dual AMS/LA tz from `last_backup_utc`) |
| `SAP_BW4HANA.html` | `GET /sap_skills/api/bw4hana_status` | `hana-backup-date` (dual AMS/LA tz) |

BW/4HANA cockpit also shows **HANA license expiry** with live seconds-precision
countdown (from `M_LICENSE.EXPIRATION_DATE`). Color:
- `> 30 days` → green
- `6–30 days` → orange
- `≤ 5 days` or expired → red

---

## Backup schedule (sapidess4 FIV)

Weekly full data backup every **Saturday 09:33 America/Los_Angeles** (computed
server-side; `hana_next_backup_utc` exposed by `/api/system_status`).

BWH has no scheduled full backup yet (manual BACKUP DATA only). Automatic log
and catalog backups run every minute by HANA, streaming to GCS via backint.

---

## References

- Google Cloud Agent for SAP: https://cloud.google.com/solutions/sap/docs/agent-for-sap/latest
- SAP HANA Backup and Recovery: https://help.sap.com/docs/SAP_HANA_PLATFORM
- Bucket: https://console.cloud.google.com/storage/browser/sap-hana-backint?project=internal-sales
- Related skills: `saporaclebackup` (Oracle DB via BR*Tools — separate flow, different bucket prefix)

---

# SQL Server (SQ1) → GCS — Two-Phase Backup

SAP on SQL Server doesn't speak the HANA BACKINT protocol, so we built a
**two-phase flow** that lands backups in the same shared bucket
(`sap-hana-backint`) under a dedicated prefix. Configured 2026-04-16 on
`sap-sql-ides` (Windows Server 2022, SID `SQ1`, SQL Server 2012 SP1).

## Architecture

```
SQL Agent / Task Scheduler
  └─ D:\Backup\scripts\run_backup.ps1 (SYSTEM)
       ├─ Phase 1: sqlcmd BACKUP DATABASE [SQ1] WITH COMPRESSION,CHECKSUM,INIT
       │            → D:\Backup\SQ1_<YYYYMMDD_HHMMSS>.bak
       └─ Phase 2: gsutil -m cp new .bak → gs://sap-hana-backint/sql_server_backup/SQ1/full/
                   then prune local to newest 2 files (safety: only delete if object exists in GCS)
```

Auth uses the same shared SA key (compute default SA `354793711235-compute@developer.gserviceaccount.com`)
as the HANA backint flow, but **gsutil on GCE prefers the VM metadata token**
which has limited scopes (read-only GCS). We force JSON-key auth via a `.boto`
config with `[GoogleCompute] service_account = none`.

## GCS layout

```
gs://sap-hana-backint/sql_server_backup/
  └─ SQ1/full/SQ1_<YYYYMMDD_HHMMSS>.bak   (compressed SQL Server native backup)
```

Room for future `diff/` and `log/` if differential / log backups are added
(currently only full backups).

## Files on sap-sql-ides

| Path | Purpose |
|------|---------|
| `D:\Backup\` | Local backup landing dir (kept: newest 2 `.bak` files) |
| `D:\Backup\scripts\phase1_backup.ps1` | SQL Server BACKUP DATABASE via sqlcmd |
| `D:\Backup\scripts\phase2_upload.ps1` | gsutil -m cp + local retention prune |
| `D:\Backup\scripts\run_backup.ps1` | Wrapper: phase1 → phase2, logs to `D:\Backup\logs\` |
| `D:\Backup\logs\run_SQ1_<ts>.log` | Per-run combined log |
| `C:\ProgramData\gcs\internal-sales-sa.json` | Shared SA key (ACL: SYSTEM+Admins+sq1adm, 600) |
| `C:\ProgramData\gcs\boto.cfg` | `service_account = none` + `gs_service_key_file = ...` |

## Environment variables (set system-wide, Machine scope)

```
BOTO_CONFIG                  = C:\ProgramData\gcs\boto.cfg
GOOGLE_APPLICATION_CREDENTIALS = C:\ProgramData\gcs\internal-sales-sa.json
```

Set via `[System.Environment]::SetEnvironmentVariable(...,'Machine')` so
scheduled tasks running as SYSTEM inherit them.

## Scheduled task

| Name | `SQ1_Weekly_Backup_GCS` |
|------|--------|
| Trigger | Weekly, Saturday 22:00 local (America/Chicago — server TZ) |
| Action | `powershell.exe -NoProfile -ExecutionPolicy Bypass -File D:\Backup\scripts\run_backup.ps1 -Database SQ1 -KeepLocal 2` |
| Principal | SYSTEM (no stored credential) |
| Execution time limit | 12 hours |
| Multiple instances | IgnoreNew |

Manage via `Get-ScheduledTask -TaskName SQ1_Weekly_Backup_GCS`,
`Start-ScheduledTask`, `Unregister-ScheduledTask -Confirm:$false`.

## Replicating the two-phase flow to another SQL Server host

1. Install Google Cloud SDK (gcloud + gsutil).
2. Copy shared SA key to `C:\ProgramData\gcs\internal-sales-sa.json` (perms:
   remove inheritance, grant only SYSTEM + Administrators + SID-admin FullControl).
3. Write `C:\ProgramData\gcs\boto.cfg`:
   ```
   [GoogleCompute]
   service_account = none
   [Credentials]
   gs_service_key_file = C:\ProgramData\gcs\internal-sales-sa.json
   ```
4. Set machine env vars `BOTO_CONFIG` + `GOOGLE_APPLICATION_CREDENTIALS`.
5. Copy the three PS1 scripts to `D:\Backup\scripts\` (parameterize `-Database`
   and `-GcsPrefix`).
6. `Register-ScheduledTask` as above.
7. First manual run: `powershell.exe -File D:\Backup\scripts\run_backup.ps1`.

## Verifying a run

```powershell
# Last run log
Get-ChildItem D:\Backup\logs\run_SQ1_*.log | Sort-Object LastWriteTime -Descending | Select-Object -First 1 | Get-Content

# Local files (should be <= KeepLocal)
Get-ChildItem D:\Backup\SQ1*.bak | Sort-Object LastWriteTime -Descending

# GCS state
gsutil ls -l gs://sap-hana-backint/sql_server_backup/SQ1/full/
```

## Troubleshooting — SQL Server flow

| Symptom | Cause | Fix |
|---------|-------|-----|
| `ResumableUploadAbortException: 403 Provided scope(s) are not authorized` | gsutil using VM metadata token (limited scope) instead of JSON key | Ensure `C:\ProgramData\gcs\boto.cfg` has `[GoogleCompute] service_account = none` + `gs_service_key_file = ...`, and `BOTO_CONFIG` env var points at it |
| `GOOGLE_APPLICATION_CREDENTIALS` set but still 403 | boto config missing — env var alone isn't enough for gsutil on GCE | Write boto.cfg (see above) |
| Detached cmd.exe batch exits after first gsutil cp completes (never writes sentinel, next upload never starts) | Known quirk of `Invoke-CimMethod Win32_Process.Create` + cmd batch when launched from a WinRM PS session — batch loses its console handle after the child process completes | For multi-file sequential uploads from a WinRM session, invoke each file in a **separate** WMI-detached batch, OR use Task Scheduler (which runs as SYSTEM with a real session) — the scheduled `run_backup.ps1` flow is not affected |
| phase1 `BACKUP DATABASE` fails rc=1 | Usually disk space on D: (need `~1.5× DB size` for compressed backup) or SQL Server service account can't write to `D:\Backup` | Check `D:\Backup\logs\phase1_*.log`; verify `NT SERVICE\MSSQL$SQ1` has write access to `D:\Backup` |
| phase2 `gsutil cp failed rc=...` | Network glitch, SA key rotated, bucket ACL changed | Re-run `phase2_upload.ps1` standalone — it's idempotent, skips files already in GCS |
| Prune deleted file not yet in GCS | Shouldn't happen — phase2 only prunes files that return RC=0 from `gsutil stat` | Still: check `D:\Backup\logs\phase2_*.log` for `PRUNE:` vs `PRUNE-SKIP:` lines |

## Initial seed upload (2026-04-16)

The two pre-existing local backups on `D:\Backup\` were uploaded during setup:
- `full_backup.bak` (193.29 GiB, 2025-10-17) → `SQ1_full_20251017_164131.bak`
- `SQ1_20260416_133723.bak` (104.75 GiB, 2026-04-16) → same name

Both remain local (user chose `KeepLocal=2`). Next run overwriting the newer
one is governed by the LastWriteTime-based pruning in phase2.
