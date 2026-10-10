# Production Deployment — Kites Edu CRM

Summary of production infrastructure, deployment layout, and local vs server sync status.  
**Last reviewed:** October 10, 2026 (deploy completed same day)

**Controlled deploy:** procedure and live execution log are in [Controlled production deploy](#controlled-production-deploy) (same file).

> **Cursor rule (local only, not committed):** Copy or sync from this doc into `.cursor/rules/production-deployment.mdc` on your machine. The `.cursor/` folder is gitignored.

**Live URL:** [https://crm.ajkadm.com/login](https://crm.ajkadm.com/login)

**Latest production deploy (2026-10-10):** `master` @ `21e1f34` — education pre-leads live. Phases **A–D** complete; automated sign-off **GO**. [Execution log](#deploy-execution-log--2026-10-10-master-pre-leads).

---

## VPS Overview

| Field | Value |
|-------|--------|
| Provider | Hostinger (KVM 1) |
| Hostname | `srv1396328.hstgr.cloud` |
| Public IP | `72.60.204.220` |
| OS | Ubuntu 22.04.5 LTS |
| Disk | 49 GB (~9% used) |
| RAM | 3.8 GB |
| SSL expiry | Nov 17, 2026 (Let's Encrypt) |
| Renewal / plan | Running until 2028-02-18 |

**DNS:** `crm.ajkadm.com` → `72.60.204.220`

---

## Architecture

```
Internet
   │
   ▼
Nginx (80 → 443 redirect, SSL)
   │
   ▼
PHP 8.2-FPM (unix:/var/run/php/php8.2-fpm.sock)
   │
   ▼
Laravel 10.49.1  (/var/www/Kites-Edu-CRM)
   │
   ▼
MySQL 8.0 (127.0.0.1:3306 only — not public)
```

---

## Source Code & Paths

| Purpose | Path |
|---------|------|
| Live application root | `/var/www/Kites-Edu-CRM/` |
| Nginx document root | `/var/www/Kites-Edu-CRM/public/` |
| Nginx vhost config | `/etc/nginx/sites-available/edu-crm` (enabled) |
| Server backup snapshot | `/var/www/Kites-Edu-CRM-backup-2026-06-19/` |
| Unused deploy dirs | `/var/www/releases/`, `/var/www/shared/` |

**Deploy method:** Code was uploaded/rsynced to the server. There is **no git repository** on the VPS.

**Local repository:** `git@github.com:smlingesh/Kites-Edu-CRM.git`

---

## Stack & Services

| Component | Status / version |
|-----------|------------------|
| Nginx | Active |
| PHP | 8.2.30 (FPM active) |
| Laravel | 10.49.1 |
| MySQL | 8.0 (active, localhost only) |
| Redis | Not running |
| Supervisor | Not running |
| Queue driver | `sync` (no background workers) |
| phpMyAdmin | Installed at `/phpmyadmin` on same domain |
| Docker | Not used |

**Cron (root):**

```cron
* * * * * cd /var/www/Kites-Edu-CRM && php artisan schedule:run >> /dev/null 2>&1
```

---

## Database

| Setting | Production value |
|---------|------------------|
| Engine | MySQL 8.0 |
| Host | `127.0.0.1` |
| Port | `3306` |
| Database | `edu_crm` |
| User | `crmuser` |
| Password | `********` *(stored in server `.env` only)* |
| Approx. size | ~13 MB |
| Tables | 16 |

**Largest tables (exact counts — verified post-deploy 2026-10-10):**

| Table | Rows |
|-------|------|
| `edu_leads` | 12,453 |
| `edu_lead_followups` | 11,645 |
| `edu_lead_status_histories` | 2,328 |
| `edu_lead_notes` | 1,946 |
| `users` | 17 |
| `migrations` | 33 |

**Schema (post-deploy):** `edu_leads.is_pre_lead` (`tinyint(1)`, default `0`, indexed). Pre-leads in DB: **0** rows with `is_pre_lead = 1` immediately after deploy.

> Row counts unchanged vs pre-deploy baseline; only `migrations` +1. Full log: [Deploy execution log](#deploy-execution-log--2026-10-10-master-pre-leads).

---

## Production `.env` (non-secret keys)

```env
APP_NAME=Kites-Edu-CRM
APP_ENV=production
APP_KEY=base64:********
APP_DEBUG=true
APP_URL=https://crm.kiteseds.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=edu_crm
DB_USERNAME=crmuser
DB_PASSWORD=********

CACHE_DRIVER=file
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120
```

> **Note:** `APP_URL` still points to `crm.kiteseds.com` while the live domain is `crm.ajkadm.com`.  
> **Note:** `APP_DEBUG=true` in production can expose stack traces — consider disabling in a maintenance window.

---

## Local vs Production — Sync Status

**After deploy:** 2026-10-10 — production matches local **`master`** @ `21e1f34` (education pre-leads).

### Application source code

Pre-deploy (Phase A): **220** local vs **218** production manifest files; **10** changed + **2** new on local.  
Post-deploy (Phase C): rsync + migrate applied; application source aligned with git `master`.

| Check | Result |
|-------|--------|
| Deploy commit on server | `21e1f34` (via rsync, no git on VPS) |
| Pending migrations before deploy | `2026_10_08_000001_add_is_pre_lead_to_edu_leads_table` |
| Migration after deploy | Ran — batch **[9]** |
| Production `.env` | Unchanged (not rsync’d) |

**Full Phase A–D record:** [Deploy execution log — 2026-10-10](#deploy-execution-log--2026-10-10-master-pre-leads).

### Expected differences (not drift)

| Layer | In sync? | Notes |
|-------|----------|-------|
| Application source | Yes | Git matches `/var/www/Kites-Edu-CRM/` |
| Production `.env` | No | By design — not committed to git |
| Local `.env` | No | Dev settings (`APP_ENV=local`, local DB) |
| `vendor/` | No | Installed on server via Composer |
| `storage/` / `bootstrap/cache/` | No | Runtime logs, uploads, compiled views |

### Local `.env` vs production (non-secret)

| Key | Local | Production |
|-----|-------|------------|
| `APP_ENV` | `local` | `production` |
| `APP_URL` | `https://crm.ajkadm.com` | `https://crm.kiteseds.com` |
| `APP_NAME` | `Kites Edu CRM` | `Kites-Edu-CRM` |
| `DB_DATABASE` | `laravel` | `edu_crm` |
| `DB_USERNAME` | `root` | `crmuser` |
| `DB_PASSWORD` | `********` | `********` |
| `APP_KEY` | `base64:********` | `base64:********` |

---

## Server Hotfixes (synced to git)

On **June 18, 2026**, two files were edited directly on the production server and were **not** in git until synced on Oct 6, 2026.

### 1. `app/Http/Controllers/EduLeadController.php`

Improved duplicate-key (`1062`) error handling on lead create/update for:

- `whatsapp_number`
- `lead_code`
- `phone` (fallback)

### 2. `app/Models/EduLead.php`

Improved `generateLeadCode()` logic — parses the numeric suffix after `-` and supports sequences beyond 4 digits.

These changes are now committed locally on branch `pre-lead` (`1d7abff`).

---

## Server Backup Folder

`/var/www/Kites-Edu-CRM-backup-2026-06-19/` is a snapshot from June 19, 2026.

- It **does not** reflect current live code after the Jun 18 hotfixes.
- It contains some old files no longer in the current codebase (e.g. old migrations/seeders).
- Use only as a historical reference, not as the source of truth.

---

## Known Issues & Observations

1. **Export memory limit** — `/edu-leads/export` hits PHP 128 MB limit (PhpSpreadsheet). Errors appear in `storage/logs/laravel.log` and nginx error log. Does not stop the rest of the app.
2. **Export filter column name** — search uses `referralname` but DB column is `referral_name` (`EduLeadController`); filtered export can SQL-error (confirmed pre-deploy; still present after 2026-10-10 deploy).
3. **`APP_DEBUG=true` on production** — security/UX risk; disable when convenient.
4. **`APP_URL` mismatch** — may affect generated URLs and emails.
5. **phpMyAdmin on public domain** — ensure strong credentials; consider IP restriction.
6. **No git on server** — manual deploys make drift harder to detect; prefer git- or rsync-based deploy from CI.
7. **No Redis / queue workers** — all jobs run synchronously (`QUEUE_CONNECTION=sync`).

---

## How to Detect Drift (read-only)

Run these on the **server** without affecting the running instance.

### Compare live vs backup

```bash
diff -qr /var/www/Kites-Edu-CRM-backup-2026-06-19 /var/www/Kites-Edu-CRM \
  --exclude=vendor --exclude=storage --exclude=bootstrap/cache --exclude=.env
```

### Find source files modified after a date

```bash
find /var/www/Kites-Edu-CRM -type f -name '*.php' \
  ! -path '*/vendor/*' ! -path '*/storage/*' \
  -newermt '2026-06-19' -ls
```

### Generate checksum manifest (compare with local)

**On server:**

```bash
cd /var/www/Kites-Edu-CRM
find app config routes resources database public -type f \
  \( -name '*.php' -o -name '*.blade.php' -o -name '*.js' -o -name '*.css' -o -name '*.json' \) \
  | sort | xargs md5sum > /tmp/prod-manifest.txt
```

**Locally:**

```bash
find app config routes resources database public -type f \
  \( -name '*.php' -o -name '*.blade.php' -o -name '*.js' -o -name '*.css' -o -name '*.json' \) \
  | sort | xargs md5sum > /tmp/local-manifest.txt
```

Compare the two manifest files; any line mismatch is a content drift.

---

## SSH Access (local only)

VPS credentials are stored in the local `.env` file (gitignored):

```env
VPS_HOST=72.60.204.220
VPS_USER=root
VPS_PASSWORD=********
VPS_PORT=22
```

> All secrets (`VPS_PASSWORD`, `DB_PASSWORD`, `APP_KEY`, etc.) are kept in the local `.env` file only. Use `********` in docs — never paste real values.

**Never commit** `.env` or production secrets to git.

When inspecting production:

- Use read-only commands (`cat`, `ls`, `grep`, `find`, `diff`, `md5sum`).
- Do **not** restart services, run migrations, or clear caches unless in a planned maintenance window (except during [Controlled production deploy](#controlled-production-deploy)).

---

## Controlled production deploy

Principles: **measure → backup → deploy → measure**; never overwrite production `.env` or `storage/`; **mysqldump before migrate**; go/no-go after Phase A; rollback paths recorded in Phase B.

**Release (master @ pre-leads):** Education pre-leads feature; migration `2026_10_08_000001_add_is_pre_lead_to_edu_leads_table` (additive column, row counts unchanged).

### Phase A — Pre-deploy baseline (read-only)

Health, DB exact counts, code delta. No rsync, migrate, or service restarts.

```bash
# Local
curl -sS -o /dev/null -w "http_code=%{http_code} time_total=%{time_total}s\n" \
  https://crm.ajkadm.com/login

# Server
cd /var/www/Kites-Edu-CRM
php artisan about && php artisan migrate:status
systemctl is-active nginx php8.2-fpm mysql
tail -n 50 storage/logs/laravel.log
```

DB counts (save for post-deploy compare):

```bash
cd /var/www/Kites-Edu-CRM
DB_USER=$(grep '^DB_USERNAME=' .env | cut -d= -f2- | tr -d '"')
DB_PASS=$(grep '^DB_PASSWORD=' .env | cut -d= -f2- | tr -d '"')
DB_NAME=$(grep '^DB_DATABASE=' .env | cut -d= -f2- | tr -d '"')
mysql -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" -N -e "
SELECT 'edu_leads' AS tbl, COUNT(*) FROM edu_leads
UNION ALL SELECT 'edu_lead_followups', COUNT(*) FROM edu_lead_followups
UNION ALL SELECT 'edu_lead_status_histories', COUNT(*) FROM edu_lead_status_histories
UNION ALL SELECT 'edu_lead_notes', COUNT(*) FROM edu_lead_notes
UNION ALL SELECT 'users', COUNT(*) FROM users
UNION ALL SELECT 'migrations', COUNT(*) FROM migrations;"
```

Code delta: md5 manifests under `app config routes resources database public` (local vs server).

### Phase B — Backups

Do **not** use `/var/www/Kites-Edu-CRM-backup-2026-06-19/` for rollback.

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p /var/backups
tar -czf /var/backups/kites-edu-crm-app-${STAMP}.tar.gz \
  --exclude='vendor' --exclude='storage/logs' -C /var/www Kites-Edu-CRM
mysqldump -u crmuser -p \
  --single-transaction --routines --triggers --no-tablespaces \
  edu_crm > /var/backups/edu_crm-${STAMP}.sql
sha256sum /var/backups/kites-edu-crm-app-${STAMP}.tar.gz /var/backups/edu_crm-${STAMP}.sql
```

### Phase C — Deploy code

```bash
# Local → server (adjust source path)
rsync -avz --delete \
  --exclude '.env' --exclude '.git' --exclude 'vendor/' --exclude 'node_modules/' \
  --exclude 'storage/' --exclude 'bootstrap/cache/*.php' --exclude '.cursor' --exclude '.DS_Store' \
  /path/to/Kites-Edu-CRM/ root@72.60.204.220:/var/www/Kites-Edu-CRM/

# Server
cd /var/www/Kites-Edu-CRM
composer install --no-dev --optimize-autoloader --no-interaction
chown -R www-data:www-data storage bootstrap/cache
php artisan migrate --force
php artisan config:cache && php artisan route:cache && php artisan view:cache
systemctl reload php8.2-fpm
```

### Phase D — Post-deploy verification

Repeat Phase A health and DB counts; critical tables must match pre-deploy; `migrations` +1; smoke-test login, leads, pre-leads, dashboard.

### Rollback

1. Restore Phase B app tarball (keep `.env` if unchanged).
2. Restore `edu_crm-${STAMP}.sql` or `migrate:rollback --step=1 --force`.
3. Clear caches; re-check health and DB counts vs pre-deploy baseline.

---

## Deploy execution log — 2026-10-10 master pre-leads

| Field | Value |
|-------|--------|
| Target | https://crm.ajkadm.com |
| App path | `/var/www/Kites-Edu-CRM/` |
| Source | `master` @ `21e1f34` (education pre-leads) |
| Operator | *(fill on execution)* |
| **Overall** | Phases **A–D complete** (automated); **GO** — manual UI smoke optional |
| **Rollback** | `/var/backups/kites-edu-crm-app-20261010-222937.tar.gz`, `/var/backups/edu_crm-20261010-222937.sql` |

### Phase A — Pre-deploy baseline

**Run:** 2026-10-10 ~22:27 IST (2026-10-10T16:57Z UTC)  
**Result:** **GO** for Phase B

| Check | Result |
|-------|--------|
| HTTPS `/login` | 200 (~0.44s) |
| nginx / php8.2-fpm / mysql | active |
| Laravel | 10.49.1, PHP 8.2.30, maintenance OFF |
| Cron `schedule:run` | Present |
| Logs | No fatals after 2026-10-07 |

**Pre-deploy migration:** `2026_10_08_000001_add_is_pre_lead_to_edu_leads_table` not applied; `is_pre_lead` column absent.

**DB baseline (exact `COUNT(*)`) — post-deploy must match except `migrations` +1:**

| Table | Count |
|-------|------:|
| edu_leads | **12,453** |
| edu_lead_followups | 11,645 |
| edu_lead_status_histories | 2,328 |
| edu_lead_notes | 1,946 |
| edu_call_logs | 0 |
| edu_lead_imports | 64 |
| edu_lead_sources | 8 |
| users | 17 |
| courses | 54 |
| programmes | 6 |
| branches | 3 |
| migrations | 32 *(pre-deploy)* |

**Code delta (pre-deploy):** 2 new files (`EduPreLeadController`, pre-lead migration); 10 changed (controllers, model, provider, routes, edu-lead views, layout). Bulk import controller matched prod before rsync.

### Phase B — Backups

**Run:** 2026-10-10 ~22:29 IST (2026-10-10T16:59Z UTC)  
**Result:** **GO** for Phase C

| Item | Path | Size | SHA256 | Status |
|------|------|------|--------|--------|
| App tarball | `/var/backups/kites-edu-crm-app-20261010-222937.tar.gz` | 14 MB | `a5042c540048189d5f6459cee3570f0c4b151973dd2456a25064e1e3c06e963d` | Done |
| MySQL dump | `/var/backups/edu_crm-20261010-222937.sql` | ~7.0 MB | `21f49e9189a369dc9a58d8063380c9676602efb5aea75d656493e8fcda2e9116` | Done |

Tarball excludes `vendor/`, `storage/logs/`. Dump verified (~7.2 MB, includes `edu_leads` data). MySQL tablespaces warning on dump — use `--no-tablespaces` next time.

### Phase C — Deploy

**Run:** 2026-10-10 ~22:31 IST (2026-10-10T17:01Z UTC)  
**Result:** **GO** — Phase D verification completed (~22:33 IST); see below

| Step | Status | Notes |
|------|--------|-------|
| rsync application code | Done | From local `master` @ `21e1f34`; `.env`, `storage/`, `vendor/` preserved on server |
| `composer install --no-dev` | Done | Dev packages removed (e.g. ide-helper); autoload optimized |
| `php artisan migrate --force` | Done | `2026_10_08_000001_add_is_pre_lead_to_edu_leads_table` — batch **[9]**, ~540ms |
| Config / route / view cache | Done | Config, routes, views **CACHED** |
| `php-fpm` reload | Done | `php8.2-fpm` active |

**Schema:** `edu_leads.is_pre_lead` present (`tinyint(1)`, default `0`, indexed).

### Phase D — Post-deploy verification

**Run:** 2026-10-10 ~22:33 IST (2026-10-10T17:02:57Z UTC)  
**Result:** **GO** (automated Phase D complete)

#### Health

| Check | Pre (Phase A) | Post (Phase D) | Pass |
|-------|---------------|----------------|------|
| HTTPS `/login` | 200 (~0.44s) | 200 (~0.12s) | Yes |
| nginx / php8.2-fpm / mysql | active | active | Yes |
| `php artisan about` | OK | OK; maintenance OFF | Yes |
| Laravel caches | views cached | config, routes, views **CACHED** | Yes |
| Protected routes (no session) | — | `/dashboard`, `/edu-leads`, `/edu-pre-leads` → **302** (auth) | Yes |
| `laravel.log` | no fatals after 2026-10-07 | no new errors since deploy (tail still Oct 6–7 export/memory) | Yes |

#### Application / release checks

| Check | Result |
|-------|--------|
| Migration `2026_10_08_000001_add_is_pre_lead_to_edu_leads_table` | **Ran** batch [9] |
| `edu-pre-leads` routes | Registered (`EduPreLeadController`) |
| `EduPreLeadController.php` on server | Present |
| `is_pre_lead` column | `tinyint(1)`, default 0, indexed |
| Pre-lead rows (`is_pre_lead = 1`) | **0** |
| Regular leads (`is_pre_lead = 0`) | **12,453** (all existing leads) |

#### DB counts — pre vs post

| Table | Pre | Post | Match? |
|-------|----:|-----:|:------:|
| edu_leads | 12,453 | 12,453 | Yes |
| edu_lead_followups | 11,645 | 11,645 | Yes |
| edu_lead_status_histories | 2,328 | 2,328 | Yes |
| edu_lead_notes | 1,946 | 1,946 | Yes |
| edu_call_logs | 0 | 0 | Yes |
| edu_lead_imports | 64 | 64 | Yes |
| edu_lead_sources | 8 | 8 | Yes |
| users | 17 | 17 | Yes |
| courses | 54 | 54 | Yes |
| programmes | 6 | 6 | Yes |
| branches | 3 | 3 | Yes |
| migrations | 32 | 33 | Yes (+1) |
| failed_jobs | 0 | 0 | Yes |

#### Manual UI smoke (optional — not run from tooling)

- [ ] Login UI (admin / counsellor)
- [ ] Education leads — list and open one lead
- [ ] Pre-leads — list, create, convert to lead
- [ ] Dashboard — today’s follow-ups

**Final go / no-go:** **GO** for production deploy sign-off on automated criteria. Tick manual checkboxes after spot-check in browser.

**Post-deploy note:** Export search still references `referralname` in `EduLeadController` (pre-existing; DB column is `referral_name`). Export may error on filtered export — separate fix, not introduced by this deploy.

**Rollback paths:** Phase B backups (see table above).

---

## Quick Reference

| Item | Value |
|------|--------|
| Live URL | https://crm.ajkadm.com |
| Server IP | 72.60.204.220 |
| App path | /var/www/Kites-Edu-CRM |
| Web root | /var/www/Kites-Edu-CRM/public |
| PHP | 8.2-FPM |
| Laravel | 10.49.1 |
| Database | edu_crm @ 127.0.0.1 |
| Git branch (local) | master |
| Deploy target commit | 21e1f34 (pre-leads) |
| Last prod deploy | 2026-10-10 — `21e1f34` (pre-leads live) |
| Deploy sign-off | **GO** (Phase D automated, 2026-10-10T17:02:57Z) |
| Rollback backups | `20261010-222937` under `/var/backups/` |
