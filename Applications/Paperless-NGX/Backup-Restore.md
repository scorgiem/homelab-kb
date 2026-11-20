# Backup & Restore — Paperless-ngx on TrueNAS SCALE

Two independent layers protect this system. Use **both**. They fail in different ways, so one covers the other.

| Layer | What it protects | Restores to |
|---|---|---|
| **A. `document_exporter`** | A portable, version-independent copy of everything: documents + all metadata (tags, types, correspondents, storage paths, workflows, custom fields, users) | A fresh Paperless install, even a different machine/version |
| **B. ZFS snapshots + replication** | Block-level copy of the datasets (archive, app data, DB, export) | The same dataset layout, fast rollback |

> **The `document_exporter` output is the one you can trust across a full rebuild.** Snapshots are faster for "undo what I did an hour ago", but a Postgres data directory restored onto a different PostgreSQL version can refuse to start. The exporter has no such dependency.

---

## Layer A — `document_exporter` (the restorable backup)

### What it does
Writes every document (original + archive PDF) plus a `manifest.json` describing all metadata into `/mnt/eve/paperless_export` (`/usr/src/paperless/export` in the container). A matching `document_importer` rebuilds a blank Paperless from it.

### Run it manually (eve shell)
```bash
sudo docker exec -it ix-paperless-ngx-paperless-1 \
  document_exporter ../export --delete --no-progress-bar
```
- `../export` resolves to `/usr/src/paperless/export` → host `/mnt/eve/paperless_export`.
- `--delete` removes files from the export that no longer exist in Paperless, keeping it a true mirror.
- Add `--zip` if you prefer a single archive per run.

### Schedule it (cron)
Create a **TrueNAS → System → Advanced → Cron Job** (or a Paperless scheduled task), daily, early morning:
```bash
docker exec ix-paperless-ngx-paperless-1 document_exporter ../export --delete --no-progress-bar
```
Run as a user that can reach Docker (root). Confirm the first few runs land files in `/mnt/eve/paperless_export`.

> **Status:** as of the last project check this cron was **not yet in place** — it is item 5 on the outstanding list. Set it up; it's the backbone of disaster recovery.

### Verify a backup is good
- `manifest.json` exists and its size grows roughly with document count.
- Spot-check that a recent document's original PDF is present in the export tree.
- Occasionally test-import into a throwaway Paperless instance (see "Restore", below).

---

## Layer B — ZFS snapshots + replication

### Snapshot tasks (Data Protection → Periodic Snapshot Tasks)
Snapshot these datasets:

| Dataset | Why | Suggested schedule / retention |
|---|---|---|
| `eve/archive_paperless` | The documents themselves | Hourly keep 24, daily keep 30, weekly keep 8 |
| `eve/apps/paperless` (config **and** db) | Search index, classifier, PostgreSQL | Daily keep 14 |
| `eve/paperless_export` | The exporter backups | Daily keep 14 |

> **Snapshot the DB and the archive on the same cadence** so a rolled-back database still matches the files on disk. A DB that references documents the archive snapshot predates will show missing-file errors.

### Replication (Data Protection → Replication Tasks)
Replicate the snapshots to a **second TrueNAS box** (you run three) and/or external media. Off-box replication is what survives the eve hardware dying. Replicate at least `archive_paperless` and `paperless_export`.

### Consistency tip
For a guaranteed-clean DB snapshot, briefly stop the Paperless app (or at least pause consumption) around the snapshot, or rely on the `document_exporter` for the authoritative copy and treat snapshots as fast local rollback.

---

## Restore procedures

### Restore 1 — roll back a recent mistake (same server)
1. Stop the Paperless app.
2. **Data Protection / dataset → Snapshots →** clone or roll back the relevant dataset to the chosen snapshot. Roll back `archive_paperless` **and** `apps/paperless` to **matching** points in time.
3. Start the app, wait out the Deploying cycle, verify a few documents open.

### Restore 2 — rebuild from `document_exporter` (new/blank Paperless)
This is the version-independent path and the one used in `Disaster-Recovery.md`.
1. Do a clean install per `Installation.md` (datasets, mounts, env vars, app Running) — a **blank** Paperless.
2. Make the export available at `/mnt/eve/paperless_export` (restore it from replication/snapshot or copy it in).
3. Import:
   ```bash
   sudo docker exec -it ix-paperless-ngx-paperless-1 \
     document_importer ../export --no-progress-bar
   ```
4. Wait for it to finish (large archives take a while — it rebuilds documents and metadata).
5. Everything from `Configuration.md` — tags, types, storage paths, workflows, correspondents, custom fields, users — comes back from the manifest. **Do not** rebuild those by hand.
6. Re-check the env vars and reverse proxy (those live in the app config, not in the export).

### Restore 3 — DB-level restore
Possible by restoring the `apps/paperless/db` dataset + the `apps/paperless/config` (index/classifier) dataset + `archive_paperless` **all to the same snapshot time**, onto the **same PostgreSQL major version** (18). If the PostgreSQL version differs, use Restore 2 instead.

---

## What is NOT in the backups (recreate by hand)

These live in TrueNAS app config / infrastructure, not in Paperless data:
- App **environment variables** (keep them in `Installation.md` §5 — treat that as the source of truth).
- **SMB shares** and dataset permissions (UID 568).
- **NPM proxy host** and Cloudflare Access config.
- **Ollama** app + pulled models.
- **Email forwarders** on Domains.co.za (the mailbox contents aren't needed; mail rules themselves are inside Paperless and *are* exported).

Keep this docs folder itself backed up alongside the exports.

---

## Recommended backup cadence (summary)

- **Daily:** `document_exporter --delete` cron → `paperless_export`.
- **Hourly/daily:** ZFS snapshots of archive + app + export.
- **Daily:** replication of those snapshots to a second box / external drive.
- **Monthly:** test-import the latest export into a throwaway instance to prove the backup restores.
