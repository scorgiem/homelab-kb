# Disaster Recovery — full rebuild on a new TrueNAS SCALE server

Scenario: **eve is gone** (dead hardware, corrupted pool, or you're migrating to new hardware). You are rebuilding Paperless-ngx from scratch and need the archive and all its structure back.

This is the master runbook. It points at the other four documents for detail.

---

## 0. What you need in hand before you start

- [ ] A recent **`document_exporter` backup** (`paperless_export` dataset contents, incl. `manifest.json`) — from replication, an external drive, or the second TrueNAS box.
- [ ] **OR** replicated **ZFS snapshots** of `archive_paperless` + `apps/paperless` (+ `apps/paperless/db`) at a matching point in time.
- [ ] This docs folder (`Installation.md`, `Configuration.md`, `Backup-Restore.md`, `Troubleshooting.md`, this file).
- [ ] Network facts: eve = `192.168.20.26`, adam = `192.168.20.20`, URL `https://paperless.mervinscorgie.xyz`, app port `30070`, Ollama port `30068`.
- [ ] Access to **NPM on adam**, **Cloudflare** (mervinscorgie.xyz), and **Domains.co.za** portal (email forwarders).

**Decide your restore path first:**
- **Path 1 — `document_exporter` (preferred, version-independent).** Use when you have the export. Survives different app/PostgreSQL versions.
- **Path 2 — ZFS snapshot/replication.** Use when you have replicated datasets and are landing on the **same PostgreSQL major version (18)**. Fastest, but version-sensitive.

---

## 1. Stand up TrueNAS SCALE + pool

1. Install TrueNAS SCALE on the new box.
2. Create the storage **pool** (named `eve` to keep every path identical; if you must rename it, search-replace `/mnt/eve/` across these docs).
3. Recreate the **datasets** from `Installation.md` §1:
   `apps/paperless/config`, `apps/paperless/db`, `archive_paperless`, `paperless_consume`, `paperless_export`, `appsdata/paperless_data/trash`.
4. Set ownership to **UID 568** on all of them.
5. Recreate the consume entity sub-folders (`Sibane`, `IronsideTechnology`, `IronsideMarketing`, `RBC`, `Personal`, `_Unknown`) and the two **SMB shares** (`Paperless-Inbox` R/W, `Paperless-Archive` read-only).

---

## 2. Restore the data

### Path 1 — from `document_exporter`
1. Put the export contents into `/mnt/eve/paperless_export` (restore from replication/external media). `chown -R 568:568` it.
2. **Leave `archive_paperless` empty** — the importer repopulates it.
3. Install the Paperless app (section 3) as a **blank** instance, then run the importer (section 4).

### Path 2 — from ZFS snapshots
1. Replicate/restore the snapshots **into** the new datasets:
   - `archive_paperless` ← archive snapshot
   - `apps/paperless/db` ← DB snapshot (**same time**)
   - `apps/paperless/config` ← index/classifier snapshot (**same time**)
2. Ensure all three are the **same snapshot moment** so the DB matches the files.
3. Install the app (section 3) pointing at these populated datasets. **Skip the importer** — the data is already there. Jump to section 5.

---

## 3. Install the Paperless app

Follow `Installation.md` §4–§7 exactly:
- Install Paperless-ngx from the community catalog.
- Set the **mounts** to the production paths (the table in `Installation.md` §4).
- Set all **environment variables** (`Installation.md` §5) — these are **not** in any backup, so they come from the doc. Critical ones: `PAPERLESS_URL` = FQDN (never 127.0.0.1), `PAPERLESS_ALLOWED_HOSTS` with **no spaces**, `PAPERLESS_CONSUMER_RECURSIVE=true`, the filename format, the AI/Ollama block.
- **Expect a long "Deploying"** (the UID-568 re-own). Don't restart/roll back mid-way.
- Wait for **Running**.

---

## 4. Import the documents (Path 1 only)

```bash
sudo docker exec -it ix-paperless-ngx-paperless-1 \
  document_importer ../export --no-progress-bar
```
This rebuilds **documents + all metadata**: tags, document types, correspondents, storage paths, **workflows**, custom fields and users. Large archives take time. When it finishes, the structure from `Configuration.md` is fully back — **do not rebuild it by hand**.

---

## 5. Rebuild the surrounding infrastructure (not in backups)

1. **Ollama** (if using AI): install the Ollama app on eve, host port `30068`, pull `gemma3:4b`, `llama3.1:8b`, `embeddinggemma`. Verify:
   ```bash
   sudo docker exec ix-paperless-ngx-paperless-1 curl -s http://192.168.20.26:30068/api/tags
   ```
   Then check **Administration → Configuration → AI** in the UI (its values override env vars).
2. **NPM proxy host on adam** for `paperless.mervinscorgie.xyz`: forward `http` → `192.168.20.26:30070`, websockets ON, wildcard cert, `client_max_body_size 100M`.
3. **Cloudflare:** put `paperless.mervinscorgie.xyz` in the Access app **"Bypass – app has own login"**.
4. **Email forwarders** on Domains.co.za: `docs.sibane@`, `docs.ironsidetech@`, `docs.marketing@`, `docs.rbc@`, `docs.mervin@` → `accounts@ironsidetech.co.za`. (Mail **rules** inside Paperless are restored by the importer; the forwarders and the mailbox are external.) Re-check the mail account's app password.
5. **Tailscale** (if used for phone access): reinstall on eve, re-authorise the phone.
6. **SMB mappings** on the laptop: `P:` and `A:`.

---

## 6. Verify (acceptance checklist)

- [ ] App status **Running**; web UI loads at the FQDN and via `192.168.20.26:30070`.
- [ ] Document count and recent documents look right; open a few originals.
- [ ] `A:\` archive share shows the entity/category tree, read-only.
- [ ] Drop one PDF into each `P:\<Entity>\` → appears in Inbox tagged correctly → files to the right folder on review.
- [ ] Send one test email per alias (`docs.*`) with a PDF → each lands in the right entity folder; mail moved to `INBOX.Processed`.
- [ ] Phone app signs in (as `mscorgie`) and uploads a test receipt.
- [ ] AI **Suggest** returns results (if enabled).
- [ ] **Re-arm backups immediately:** `document_exporter` cron + ZFS snapshots + replication (`Backup-Restore.md`). A freshly restored system with no backups is one incident from the same disaster.

---

## 7. Recovery-time expectations

| Step | Rough time |
|---|---|
| TrueNAS install + pool + datasets + shares | 1–2 h |
| App install + env vars + first Deploying | 0.5–1 h (the UID-568 re-own is slow) |
| `document_importer` (Path 1) | minutes to hours by archive size |
| Snapshot restore (Path 2) | minutes to hours by archive size + link speed |
| Infra (NPM, Cloudflare, Ollama, email, Tailscale) | 1–2 h |

**Biggest risks to a smooth recovery:** no current `document_exporter` cron (fix that now); PostgreSQL version mismatch on a DB-level restore (use Path 1 instead); forgetting the env vars are doc-only (keep `Installation.md` current).
