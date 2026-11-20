# Lessons Learned — Paperless-ngx on TrueNAS SCALE

The decisions that proved right, the traps that cost time, and what to do differently on the rebuild. Read this before `Installation.md` so you build with the hindsight already applied.

---

## Architecture decisions that worked

- **Entity tag as the single key.** Making each entity workflow trigger on *Document Added + has tag* (not on each channel separately) collapsed 15 would-be workflows into 5 and made every channel — drop folder, email, phone, web — file correctly as long as it sets the tag. This is the single best structural decision in the build.
- **One storage path per entity, driven by Document Type.** A Jinja map of `type → folder` with a `99 General/Unsorted` fallback means adding a category later is a one-line change, and Paperless re-files automatically. No per-folder rules to maintain.
- **Paperless owns the folders; Windows is read-only.** Numbered categories (`01 Sales`…) + ISO-date filenames give a clean, stable Explorer view without ever risking the DB↔file link. Keeping the archive share read-only is non-negotiable.
- **Separate consume sub-folders per entity** (no-space dataset names: `IronsideTechnology` etc.) with `PAPERLESS_CONSUMER_RECURSIVE=true` — simple, scanner-friendly intake.

---

## Traps that cost time (don't repeat)

### 1. The UID-568 re-own makes every deploy slow
v3.5.x runs `usermod --uid 568 paperless` on container start, re-owning everything under `/usr/src/paperless` and forcing a ZFS image copy (processes in `D` state). **Every** install/update/settings-save means 10–20 min of "Deploying". The real cost was panic — it *looks* hung. **Lesson:** treat a long Deploying as normal, read the log before acting, and **never restart or roll back mid-way** (it restarts the slow job from zero).

### 2. `PAPERLESS_URL=127.0.0.1` broke the app
After the 3.4.2→3.5.0 auto-update, `PAPERLESS_URL` ended up at `http://127.0.0.1:30070` and `ALLOWED_HOSTS` had stray spaces → broken links and rejected requests. **Lesson:** `PAPERLESS_URL` must be the FQDN; no spaces in `ALLOWED_HOSTS`; **re-check env vars after every version bump**.

### 3. The documented dataset plan didn't match reality
The original Archive Strategy assumed `/mnt/eve/documents/paperless/{media,consume,export}`. The app actually mounted `/mnt/eve/archive_paperless`, `/mnt/eve/paperless_consume`, `/mnt/eve/paperless_export`, with config under `/mnt/eve/apps/paperless`. **Lesson:** trust `docker inspect` over the plan. `Installation.md` now documents the **real** paths — use those.

### 4. Phone uploads had no entity → mis-filed
The first phone receipt arrived typed `Receipt` with expense tags but **no entity tag**, so it couldn't be filed, and tagging it afterwards didn't move it (add-time workflows only fire once). **Lesson / fix:** the five **"Re-file on tag" (Document Updated)** workflows now move a doc whenever an entity tag is added later; and you pick the entity on the upload screen.

### 5. Overlapping mail runs → duplicate imports
Two mail checks ~coinciding imported the same attachments twice. **Lesson:** every rule moves mail to `INBOX.Processed`, rely on the `*/10` cron, and don't manually trigger a check while one is running.

### 6. Ollama reached at the wrong address
AI couldn't connect because it was pointed at `localhost:11434` / an old adam endpoint. Separate apps must talk over the LAN address `192.168.20.26:30068`. **Lesson:** cross-app calls on TrueNAS use the host LAN IP + the app's host port, never the in-container port; and UI AI settings override env vars.

### 7. Accidental objects accumulated
Stray custom fields ("Ironside Marketing", "Riverside Beauty Care", "Property"), duplicate correspondents ("Copper Mug" vs "The Coppermug"), and a broken document #1 with a missing original + an orphan `CreateObjects.ps1` in the media root. **Lesson:** entities are **tags not fields**; keep correspondent matching on Auto and merge duplicates; keep non-document files out of the media root; run `document_sanity_checker` periodically.

---

## The biggest outstanding risk

**Backups were never fully armed.** The `document_exporter` cron, ZFS snapshots and replication for `archive_paperless` / `apps/paperless` / `paperless_export` were still on the to-do list at last check. A document system with no tested backup is the real disaster waiting to happen. **On the rebuild, set backups up on day one and test a restore once** — before trusting the system with real documents. See `Backup-Restore.md`.

---

## Do-differently list for the rebuild

1. **Arm backups first**, not last: exporter cron + snapshots + off-box replication, then prove a test-import works.
2. **Build from the real paths** in `Installation.md`, not the original dataset plan.
3. **Create all five "Re-file on tag" workflows up front**, alongside the entity workflows — don't wait for a mis-filed phone receipt to discover the gap.
4. **Keep env vars in the doc as the source of truth** (they're not in any backup) and re-verify them after every app update.
5. **Sort the document list by "Added"** as your default view, so phone/email uploads (which sort by document date) don't hide.
6. **Decide remote access deliberately:** Tailscale on eve + phone is the reliable answer for the mobile app; the LAN NPM route has been flaky (openresty error on home Wi-Fi). Put `client_max_body_size 100M` and websockets on the NPM host regardless.
7. **Expect and budget for the slow Deploying** on every change — plan edits in batches so you ride out the re-own once, not five times.

---

## Reference facts to carry forward

- Host **eve** `192.168.20.26`; app server **adam** `192.168.20.20` (AdGuard, NPM, Cloudflare).
- App port **30070**; Ollama port **30068**.
- URL `https://paperless.mervinscorgie.xyz`.
- App runs as **UID 568**.
- Entities: Sibane, Ironside Technology, Ironside Marketing, Riverside Beauty Care (RBC), Personal.
- Email: `accounts@ironsidetech.co.za` on Domains.co.za, IMAP `mail.ironsidetech.co.za:993` SSL, forwarders `docs.*`.
- Stack: Paperless-ngx 3.5.0, PostgreSQL 18, Redis, Tika, Gotenberg, + Ollama.
