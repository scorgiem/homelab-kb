# Troubleshooting — Paperless-ngx on TrueNAS SCALE

Real issues hit during this project and how they were resolved. Ordered roughly by how likely you are to meet them on a rebuild.

---

## App stuck on "Deploying" / "Starting" after install, update or a settings save

**This is usually normal, not a failure.** In v3.5.x the init step maps the app user with `usermod --uid 568 paperless`, which re-owns everything under `/usr/src/paperless`. On ZFS that forces Docker to copy image files and the process sits in `D` (uninterruptible) state. It can take **10–20 minutes**, longer after a version bump.

How to tell it's working vs. stuck:
```bash
# which file is it churning through, and is it still writing?
sudo docker inspect ix-paperless-ngx-paperless-1 --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'
sudo grep -E 'write_bytes' /proc/<pid>/io     # write_bytes climbing = progressing
sudo ls -l /proc/<pid>/fd | tail -5           # shows the file currently open
```
**Do not restart or roll back mid-way** — it restarts the same slow job from zero. Wait for **Running**.

Check the container log (**Apps → paperless-ngx → Workloads → paperless → log icon**). The supporting containers (postgres, redis, gotenberg, tika) should be **Running**; `postgres_upgrade` showing **Exited** is a normal one-off job. If only `paperless` is **Starting**, it's the slow init above.

In the log:
| Log shows | Meaning | Action |
|---|---|---|
| `Applying documents.00xx…` migration lines, still advancing | 3.5.0 DB upgrade in progress | Wait (10–20 min) |
| `Mapping UID for paperless to 568` then long pause | The slow `usermod` re-own | Wait |
| `Permission denied` on `/usr/src/paperless/{data,media,consume}` | App UID ≠ dataset owner | `chown -R <AppUID>:<AppUID>` the datasets |
| Same error repeats, container restarts (new timestamp each time) | Config problem (often an env var) | Read the lines just **before** each restart |
| `Listening at: http://0.0.0.0:8000` but still "Starting" | App is up; TrueNAS health check failing | Fix `PAPERLESS_URL` (below) |

---

## Live outage 18 Nov — root cause reference

After the auto-update **3.4.2 → 3.5.0**, the app hung on Deploying. Two things were wrong:
1. The slow `usermod` re-own was mid-flight (expected; just needed time).
2. `PAPERLESS_URL` had been left at `http://127.0.0.1:30070`, and `PAPERLESS_ALLOWED_HOSTS` contained spaces.

**Fix applied:** stop app → set `PAPERLESS_URL=https://paperless.mervinscorgie.xyz` → strip spaces from `PAPERLESS_ALLOWED_HOSTS` → restore AI/Ollama vars → start → wait out the Deploying cycle. Lesson: after any version bump, **re-check env vars before assuming a crash**, and don't roll back before reading the log.

---

## `PAPERLESS_URL` set to 127.0.0.1 → broken links / rejected requests

Symptoms: LAN address behaves oddly, generated links point to localhost, requests through the domain get rejected. Set `PAPERLESS_URL` to the FQDN `https://paperless.mervinscorgie.xyz`, ensure `PAPERLESS_CSRF_TRUSTED_ORIGINS` includes both the FQDN and `http://192.168.20.26:30070`, save, wait for redeploy.

---

## LAN route returns an openresty error (phone app fails on home Wi-Fi)

The path is **AdGuard (DNS) → NPM on adam → eve:30070**. On the LAN the mobile app can get an openresty error for `paperless.mervinscorgie.xyz`, while the **same app works fine on mobile data**.

Check the NPM proxy host for this domain:
- Forward scheme **http**, host `192.168.20.26`, port `30070`
- **Websockets ON**
- Valid wildcard cert for `*.mervinscorgie.xyz`
- `client_max_body_size 100M;` in the advanced config

If the LAN path stays unreliable, use **Tailscale** (eve + phone) and point the app at the Tailscale address. The USG can't run WireGuard natively.

---

## Large scans fail to upload

Add `client_max_body_size 100M;` to the NPM proxy host. Without it, big multi-page scans are rejected by the proxy before reaching Paperless.

---

## Files dropped over SMB aren't consumed

inotify can miss SMB writes. Add env var `PAPERLESS_CONSUMER_POLLING=30` so Paperless polls the consume folder every 30 s. Confirm `PAPERLESS_CONSUMER_RECURSIVE=true` so entity sub-folders are watched.

---

## Phone receipt filed with no entity → stuck at media root / wrong folder

Phone uploads arrive via the "App upload" workflow with `Inbox` + `Scan` + type `Receipt`, but **no entity tag**, so the entity (add-time) workflow can't file them.

- **Fix for an existing doc:** open it, add the entity tag, remove `Inbox`. The **"Re-file on tag"** (Document Updated) workflow moves it (e.g. to `Ironside Technology\02 Purchases\Receipts & Expenses\2026\`).
- **Going forward:** pick the entity tag on the upload screen.
- **Finding new phone uploads:** they sort by the slip's date, not upload time, so sort the list by **Added** (newest first), or filter by the **Inbox** tag.

---

## AI "Suggest" doesn't work / Ollama connection errors

Paperless and Ollama are **separate apps**. Paperless must reach Ollama at **`http://192.168.20.26:30068`**, not `localhost:11434` (that only resolves inside the Ollama container).

```bash
sudo docker exec ix-paperless-ngx-paperless-1 curl -s http://192.168.20.26:30068/api/tags
```
Expect JSON listing `gemma3:4b`, `llama3.1:8b`, `embeddinggemma`. If it errors: check the endpoint/port, and check **Administration → Configuration → AI** in the UI — values saved there **override** the env vars (an old endpoint pointing at adam was the culprit historically). First suggestion after idle takes 20–60 s (model load). To disable AI entirely: `PAPERLESS_AI_ENABLED=false`.

---

## Duplicate imports from email

Two mail checks overlapping imported the same attachments twice. Prevent it with: "move to `INBOX.Processed`" on every rule, the `*/10` cron, and not manually triggering a check while one runs. Clean up by deleting the higher document ID of each pair.

---

## Nightly integrity/sanity check failing — missing original

Document #1 "BBBEE Certificate - Template" lost its original file and thumbnail; it had been saved with `none` as correspondent in the filename and the move failed. **Fix:** re-upload the source and delete the broken entry. Also remove any stray non-document file from the media root (an orphan `CreateObjects.ps1` was found there — delete it).

---

## Duplicate correspondents

Near-identical names split a correspondent's history ("Copper Mug" vs "The Coppermug"). Reassign documents to the canonical one, then delete the duplicate. Keeping **Matching: Auto** reduces new ones over time.

---

## Stray custom fields

"Ironside Marketing", "Riverside Beauty Care", "Property" were created as string custom fields by accident. Delete them — entities are **tags**, not fields.

---

## Quick diagnostic commands (eve shell)

```bash
# mounts the container actually has
sudo docker inspect ix-paperless-ngx-paperless-1 --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'

# tail the app log
sudo docker logs --tail 80 ix-paperless-ngx-paperless-1

# run paperless management commands (document_exporter, document_sanity_checker, etc.)
sudo docker exec -it ix-paperless-ngx-paperless-1 document_sanity_checker
```
