# Installation — Paperless-ngx on TrueNAS SCALE

**Target host:** `eve` (192.168.20.26) · **URL:** https://paperless.mervinscorgie.xyz
**App version at time of writing:** Paperless-ngx v3.5.0 (TrueNAS community catalog app 1.6.83)
**Stack:** Paperless-ngx + PostgreSQL 18 + Redis + Tika + Gotenberg, plus Ollama (separate app) for AI suggestions.

This document assumes a **complete rebuild from scratch on a new TrueNAS SCALE server**. Follow it top to bottom. Nothing here depends on the old machine being alive except the backup you restore from (see `Backup-Restore.md` / `Disaster-Recovery.md`).

---

## 0. Before you start — decisions baked into this build

- Paperless **owns** the archive folders. Windows gets a **read-only** SMB view. Never rename/move files in Explorer — it breaks the DB link. All changes happen in the Paperless web UI.
- The app runs as **UID 568** (the TrueNAS `apps` user) in v3.5.x. Every dataset it touches must be owned by / writable by UID 568.
- Five entities: **Sibane · Ironside Technology · Ironside Marketing · Riverside Beauty Care (RBC) · Personal**.
- TrueNAS dataset names cannot contain spaces, so the consume sub-folders are `Sibane`, `IronsideTechnology`, `IronsideMarketing`, `RBC`, `Personal`.

---

## 1. Create the pool and datasets

On the new server, create your pool (the old one was `eve`). Then create these datasets. **The real layout that worked in production differs from the original "documents/paperless" plan** — use the paths below, they are what the running app actually mounted (confirmed by `docker inspect`).

| Dataset (host path) | Purpose |
|---|---|
| `/mnt/eve/apps/paperless/config` | Paperless data dir: search index, classifier model, ML files |
| `/mnt/eve/apps/paperless/db` | PostgreSQL data |
| `/mnt/eve/archive_paperless` | **The archive** — originals + archived (OCR'd) copies. This is the crown jewels. |
| `/mnt/eve/paperless_consume` | Drop/consume folder (watched) |
| `/mnt/eve/paperless_export` | `document_exporter` output (the restorable backup) |
| `/mnt/eve/appsdata/paperless_data/trash` | Paperless trash (deleted-doc hold area) |

Recommended dataset settings:
- Record size: default (128K) is fine for mixed PDFs.
- Enable **snapshots** on at least `archive_paperless`, `apps/paperless` and `paperless_export` (see `Backup-Restore.md`).
- Case sensitivity: default (sensitive). SMB share handles Windows clients.

### Permissions
Give **UID 568** full control of every dataset above. Fast path from the shell:

```bash
sudo chown -R 568:568 /mnt/eve/apps/paperless \
  /mnt/eve/archive_paperless \
  /mnt/eve/paperless_consume \
  /mnt/eve/paperless_export \
  /mnt/eve/appsdata/paperless_data/trash
```

> If you later see `Permission denied` on `/usr/src/paperless/...` in the logs, the app's UID and the dataset owner disagree. Re-run the chown with the UID shown under the app's **Run As Context**.

---

## 2. Consume sub-folders (entity drop folders)

Inside the consume dataset, create one sub-folder per entity plus a catch-all. These are **folders inside the dataset**, not datasets:

```
/mnt/eve/paperless_consume/
├── Sibane/
├── IronsideTechnology/
├── IronsideMarketing/
├── RBC/
├── Personal/
└── _Unknown/        ← no entity rule; you classify these in the Inbox
```

`chown -R 568:568` the consume tree again after creating these.

---

## 3. SMB shares

Create two SMB shares:

| Share name | Path | Access |
|---|---|---|
| `Paperless-Inbox` | `/mnt/eve/paperless_consume` | **Read/write** for your SMB user |
| `Paperless-Archive` | `/mnt/eve/archive_paperless` | **Read-only** for your SMB user |

On the laptop, map:
- `P:` → `\\192.168.20.26\Paperless-Inbox`
- `A:` → `\\192.168.20.26\Paperless-Archive`

Point scanners / phone "scan to folder" at the matching `P:\<Entity>\` sub-folders.

> Keep the archive share read-only. It is the single most important rule of this whole system.

---

## 4. Install the app (TrueNAS community catalog)

1. **Apps → Discover Apps → Paperless-ngx → Install.**
2. Set the admin user. In production this is the superuser; your day-to-day login is a non-admin user (`mscorgie`). Create that second user after first boot.
3. **Storage / mounts** — set host-path mounts to match the running production layout exactly:

| Host path | Mount (container) path |
|---|---|
| `/mnt/eve/apps/paperless/config` | `/paperless/data` |
| `/mnt/eve/archive_paperless` | `/paperless/media` |
| `/mnt/eve/paperless_consume` | `/usr/src/paperless/consume` |
| `/mnt/eve/paperless_export` | `/usr/src/paperless/export` |
| `/mnt/eve/appsdata/paperless_data/trash` | `/paperless/trash` |

   PostgreSQL's data goes to `/mnt/eve/apps/paperless/db` via the app's built-in Postgres storage setting.
4. **Web port:** `30070` (host). The app is fronted by the FQDN via reverse proxy (section 6).

> **Expect a long "Deploying" after every install, update, or settings save.** On first start and on each container recreate, the init runs `usermod --uid 568 paperless`, which re-owns everything under `/usr/src/paperless` and makes Docker copy the image layer onto ZFS (processes sit in `D` state). This is normal and can take 10–20 minutes. **Do not restart or roll back mid-way** — it just restarts the same slow job from zero.

---

## 5. Environment variables

Set these under **App → Edit → Additional Environment Variables**. These are the production values.

```
PAPERLESS_URL=https://paperless.mervinscorgie.xyz
PAPERLESS_CSRF_TRUSTED_ORIGINS=https://paperless.mervinscorgie.xyz,http://192.168.20.26:30070,http://127.0.0.1:30070
PAPERLESS_ALLOWED_HOSTS=paperless.mervinscorgie.xyz,192.168.20.26,localhost,127.0.0.1
PAPERLESS_TIME_ZONE=Africa/Johannesburg
PAPERLESS_OCR_LANGUAGE=eng
PAPERLESS_CONSUMER_RECURSIVE=true
PAPERLESS_FILENAME_FORMAT={{ created_year }}/{{ created }} {{ correspondent }} - {{ title }}
PAPERLESS_FILENAME_FORMAT_REMOVE_NONE=true
PAPERLESS_CONSUMER_ENABLE_BARCODES=true
PAPERLESS_EMAIL_TASK_CRON=*/10 * * * *

# AI suggestions via Ollama on eve (optional — see section 7)
PAPERLESS_AI_ENABLED=true
PAPERLESS_AI_LLM_BACKEND=ollama
PAPERLESS_AI_LLM_ENDPOINT=http://192.168.20.26:30068
PAPERLESS_AI_LLM_MODEL=gemma3:4b
PAPERLESS_AI_LLM_EMBEDDING_BACKEND=ollama
PAPERLESS_AI_LLM_EMBEDDING_MODEL=embeddinggemma:latest
PAPERLESS_AI_LLM_REQUEST_TIMEOUT=300
```

Notes / gotchas:
- **No spaces** in `PAPERLESS_ALLOWED_HOSTS`. A stray space breaks host validation.
- `PAPERLESS_URL` **must** be the FQDN, not `127.0.0.1`. If it's left at `http://127.0.0.1:30070`, generated links point to the wrong place and requests through the domain may be rejected. (This caused a live outage on 18 Nov — see `Troubleshooting.md`.)
- The `PAPERLESS_FILENAME_FORMAT` here is only the **fallback** used when a document has no storage path. Correctly classified docs are filed by their storage path instead (see `Configuration.md`). Fallback files land at the media root — that's what the `_Unsorted` check watches for.
- If files dropped over SMB aren't picked up, add `PAPERLESS_CONSUMER_POLLING=30` (inotify sometimes misses SMB writes).
- `PAPERLESS_OCR_LANGUAGE=eng+afr` if you start receiving Afrikaans documents.
- Settings saved in the UI under **Administration → Configuration → AI** override these AI env vars.

Save → wait out the long Deploying cycle → confirm **Running**.

---

## 6. Reverse proxy and remote access

The LAN path is **AdGuard (DNS) → Nginx Proxy Manager on `adam` (192.168.20.20) → eve:30070**.

NPM proxy host for `paperless.mervinscorgie.xyz`:
- **Forward:** scheme `http`, host `192.168.20.26`, port `30070`
- **Websockets support:** ON (required for live UI updates)
- **SSL:** wildcard cert for `*.mervinscorgie.xyz`
- **Advanced / custom Nginx:** `client_max_body_size 100M;` (large scans fail to upload without this)

Cloudflare:
- `paperless.mervinscorgie.xyz` is placed in the Cloudflare Access application **"Bypass – app has own login"** so Access doesn't block the mobile app's own login flow.

Remote access for the phone app:
- Works over mobile data through the FQDN.
- For reliable off-home access, **Tailscale** (TrueNAS catalog app on eve + app on the phone) is the recommended path. The UniFi USG gateway can't run WireGuard directly.

> Known issue: on the home Wi-Fi/LAN route the mobile app can hit an openresty error from NPM. See `Troubleshooting.md` → "LAN route returns openresty error".

---

## 7. Ollama (AI suggestions) — optional

Ollama runs as its **own** TrueNAS app on eve, host port **30068**. Models pulled: `gemma3:4b`, `llama3.1:8b`, `embeddinggemma`.

Paperless reaches Ollama over eve's LAN address, **not** `localhost:11434` — the two apps are separate containers. Use `http://192.168.20.26:30068`. Port 11434 only works *inside* the Ollama container.

Verify from the eve shell once both apps are up:

```bash
sudo docker exec ix-paperless-ngx-paperless-1 curl -s http://192.168.20.26:30068/api/tags
```

You should get JSON listing the three models. Then in Paperless: **Administration → Configuration → AI**, confirm the endpoint, and open any document → **Suggest**. First call takes 20–60 s while the model loads.

If you don't want AI, set `PAPERLESS_AI_ENABLED=false` and skip this entirely — it has no effect on filing.

---

## 8. First-boot smoke test

1. Log in at https://paperless.mervinscorgie.xyz as the superuser; create the day-to-day user `mscorgie`.
2. Drop one PDF into `P:\IronsideTechnology\`. Within a cycle it should appear in the Inbox tagged `Ironside Technology`.
3. Confirm `A:\Ironside Technology\...` shows the filed copy (read-only).
4. Then move on to `Configuration.md` to rebuild tags, document types, storage paths, workflows and email.

Installation is complete once the app is **Running**, the shares mount, and a test drop files correctly.
