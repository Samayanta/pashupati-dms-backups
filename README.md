# StockFlow IMS — Daily Encrypted Backups

**Repository:** `Samayanta/pashupati-dms-backups` (public)

Dumps are **AES-256 encrypted** (OpenSSL-compatible format) — the repo being public exposes no data.

## How it works

- **Vercel Cron** (`/api/cron/backup` in the app, runs daily 17:45 UTC = 23:30 NPT)
- The route dumps all 28 public tables as SQL INSERTs (~11.4k rows), encrypts with AES-256-CBC + PBKDF2 (10k iterations), and pushes to `dumps/YYYY-MM-DD.sql.enc` via the GitHub Contents API
- Retention: files older than 30 days are auto-deleted

## Why not GitHub Actions?

The Supabase direct DB hostname (`db.<project>.supabase.co`) is **IPv6-only** (no A record). GitHub-hosted runners have no IPv6 egress. Vercel functions (where the app already runs) reach it fine.

## Secrets (Vercel project env — `vercel env`)

| Variable | Value |
|---|---|
| `CRON_SECRET` | Random hex; Vercel Cron sends it as `Authorization: Bearer <value>` |
| `BACKUP_GITHUB_TOKEN` | GitHub PAT with `repo` scope (contents write) |
| `BACKUP_PASSPHRASE` | Encryption passphrase |
| `BACKUP_REPO` | `Samayanta/pashupati-dms-backups` |

## Restore procedure (disaster drill)

Prerequisites: PostgreSQL 14+ tools (`psql`), the passphrase.

```bash
# 1. Download the dump
curl -LO "https://github.com/Samayanta/pashupati-dms-backups/raw/main/dumps/2026-08-03.sql.enc"

# 2. Decrypt (openssl-compatible format)
openssl enc -aes-256-cbc -d -salt -pbkdf2 -pass "env:BACKUP_PASSPHRASE" \
  -in 2026-08-03.sql.enc -out 2026-08-03.sql

# 3. Restore into the target database (schema must already exist; data-only dump)
psql "$DATABASE_URL" -f 2026-08-03.sql
```

### Verify restore

```sql
SELECT count(*) FROM products;              -- expect 192
SELECT count(*) FROM stock_transactions;    -- expect ~3,100
SELECT count(*) FROM notifications;         -- expect ~310
```

## Manual trigger

```bash
curl -H "Authorization: Bearer $CRON_SECRET" \
  https://pashupati-dms.vercel.app/api/cron/backup
```
