# SAP Database Backups to GCS

Claude Code skill + reference docs for SAP database backups to the shared Google Cloud Storage bucket `sap-hana-backint`, covering three systems:

- **SAP HANA** via the SAP BACKINT protocol (Google Cloud Agent for SAP)
  - `sapidess4` — S/4HANA 2023, tenants FIV (primary) + PIT (HANA Cockpit DBA)
  - `bw4hana` — BW/4HANA, tenant BWH
- **SAP on SQL Server** via a two-phase flow (native `BACKUP DATABASE` + `gsutil cp`)
  - `sap-sql-ides` — SAP ECC 6.0 IDES, SID SQ1, SQL Server 2012
- **SAP on Oracle** via BR*Tools (`brbackup` / `brarchive`) + `gsutil`
  - `sapidesecc8` — SAP ECC 6.0, SID ABA, Oracle 19c

All three land backups under distinct prefixes in `gs://sap-hana-backint/`.

## Claude Code skill install

```bash
mkdir -p ~/.claude/commands
curl -fsSL -o ~/.claude/commands/backint.md \
  https://raw.githubusercontent.com/fivetran/sap-database-backups-gcs/main/backint.md
```

Then in Claude Code: `/backint`.

## What the skill covers

- HANA BACKINT architecture, wrapper script, `hdbbackint` symlink, `parameters.json`, `global.ini [backup]`
- Per-system configuration tables for sapidess4 (S4H/FIV + PIT) and bw4hana (BWH)
- Step-by-step replication of BACKINT to a new HANA system (online `ALTER SYSTEM … WITH RECONFIGURE`)
- `google_cloud_sap_agent` CLI helpers (`installbackint`, `configurebackint`, `status -f=backint`, `backint -f=diagnose`)
- Full `parameters.json` reference — performance tuning, encryption, retention, constraint matrix
- Multiple-parameter-files pattern (separate files for data / log / catalog)
- SQL Server two-phase backup flow — `phase1_backup.ps1`, `phase2_upload.ps1`, `run_backup.ps1`, Windows Task Scheduler `SQ1_Weekly_Backup_GCS`, boto.cfg workaround for GCE VM metadata scope
- Verification SQL, troubleshooting tables per system, file/path cheat sheet

## Rendered documentation

[SAP Database Backups to GCS (HTML)](https://sapidesecc8.fivetran-internal-sales.com/sap_skills/docs/SAP_Database_Backups_to_GCS.html)

## Related skills

- `saporaclebackup` — Oracle BR*Tools specifics (separate skill)
- `hanacockpit` — SAP HANA Cockpit / XSA administration on sapidess4
- `sq1-license` — SAP license renewal on the SQ1 IDES system

## Contact

Fivetran SAP Specialist team.
