# StockFlow IMS — Nightly Encrypted Backups

**Repository:** `Samayanta/pashupati-dms-backups` (public)

The dump file is **AES-256 encrypted** — the repo being public exposes no data.

## What runs nightly

- 17:30 UTC (23:15 NPT): `pg_dump` of the live Supabase PostgreSQL database
- Encrypted with `openssl enc -aes-256-cbc -salt -pbkdf2`
- Stored as `dumps/YYYY-MM-DD.dump.enc`
- Retention: 30 days (older dumps auto-deleted)

## Secrets (set in repo settings → Secrets and variables → Actions)

| Secret | Value |
|---|---|
| `DATABASE_URL` | Full `postgres://...` pooler connection string from the app's `.env` |
| `DUMP_PASSPHRASE` | Encryption passphrase (generated below) |

## Restore procedure (disaster drill)

Prerequisites: PostgreSQL 14+ tools (`pg_restore`), the passphrase.

```bash
# 1. Download the dump
curl -LO "https://github.com/Samayanta/pashupati-dms-backups/raw/main/dumps/2026-08-03.dump.enc"

# 2. Decrypt (prompts for passphrase)
openssl enc -aes-256-cbc -d -salt -pbkdf2 -pass "env:DUMP_PASSPHRASE" \
  -in 2026-08-03.dump.enc -out 2026-08-03.dump

# 3. Restore into a target database (create it first)
createdb stockflow_restore
pg_restore --no-owner --no-privileges -d stockflow_restore 2026-08-03.dump
```

### Verify restore

```sql
SELECT count(*) FROM products;        -- expect 192
SELECT count(*) FROM stock_transactions;  -- expect ~3,100
```

## Manual trigger

`gh workflow run backup.yml -R Samayanta/pashupati-dms-backups`
