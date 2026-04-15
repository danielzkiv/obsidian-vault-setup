# VPS Claude — Linux Client Setup & Operations

> **Paired file:** [`local-claude.md`](./local-claude.md) — runbook for the Claude instance on the Linux client. Read it first so you know what the other side will do and what to hand back.

Self-contained runbook for the Claude instance on the **VPS** when onboarding or maintaining a Linux read-only mirror. The VPS side is identical regardless of client OS — this file is a Linux-client-flavored copy of `windows/vps-claude.md` for symmetry so the folder pair is self-contained.

---

## 1. Architecture & role model

Set up Syncthing on the headless VPS to sync `/root/obsidian-vault/` with a Linux desktop/laptop running Obsidian Desktop. Claude's memory is symlinked into the vault.

**Role model:** VPS is the **sole writer**. The Linux client is a **read-only mirror**.
- VPS folder type: `sendonly`.
- Client folder type: `receiveonly`.
- Rationale: Claude on the VPS is the main author. Read-only clients prevent two-writer conflicts.

---

## 2. VPS-side invariants

| Setting | Required value | Why |
|---|---|---|
| Folder `obsidian-vault` type | `sendonly` | VPS-only writes propagate out |
| `.stignore` content | `.obsidian/workspace*`, `.obsidian/cache*`, `.obsidian/graph.json`, `.obsidian/app.json`, `.trash/` | Propagates to clients; keeps Obsidian UI state from flagging as Local Additions |
| Web UI bind | `127.0.0.1:8384` only | Never exposed |
| Versioning | `simple`, keep 5 | Safety net |
| New devices | `autoAcceptFolders: false` | Prevents surprise-accept |

---

## 3. Install Syncthing on the VPS (first-time)

1. Add Syncthing's apt repo + signing key to `/etc/apt/keyrings/syncthing-archive-keyring.gpg` and `/etc/apt/sources.list.d/syncthing.list`.
2. `apt update && apt install syncthing` → pulls v1.30+.
3. `systemctl enable --now syncthing@root.service`.
4. Verify: `systemctl is-active syncthing@root` → `active`; `ss -tlnp | grep -E ':8384|:22000'` → both listeners.

---

## 4. Configure Syncthing on the VPS

```bash
CONFIG=/root/.local/state/syncthing/config.xml
API=$(grep -oP '<apikey>\K[^<]+' "$CONFIG")
BASE=http://127.0.0.1:8384/rest
```

1. Bind UI to loopback — `127.0.0.1:8384` (default).
2. `PATCH /rest/config/gui` → set `admin` user + password (bcrypted server-side).
3. `DELETE /rest/config/folders/default` → remove the unused auto-created folder.
4. Add the vault as `sendonly`:
   ```bash
   curl -sX POST -H "X-API-Key: $API" -H "Content-Type: application/json" \
     -d '{"id":"obsidian-vault","label":"Obsidian Vault","path":"/root/obsidian-vault",
          "type":"sendonly","fsWatcherEnabled":true,"fsWatcherDelayS":5,
          "versioning":{"type":"simple","params":{"keep":"5"}},"devices":[]}' \
     "$BASE/config/folders"
   ```
5. Write `.stignore` in `/root/obsidian-vault/`:
   ```
   .obsidian/workspace*
   .obsidian/cache*
   .obsidian/graph.json
   .obsidian/app.json
   .trash/
   ```
6. No restart needed.

---

## 5. Firewall & connectivity

1. TCP 22000 reachable from the internet (Syncthing sync port). `ufw allow 22000/tcp` if UFW is active.
2. Leave 21027/udp closed (LAN discovery, irrelevant over the internet).
3. Confirm `globalAnnounceEnabled: true` + `relaysEnabled: true` via `GET /rest/config/options`.
4. Print VPS Device ID: `syncthing --device-id --home=/root/.local/state/syncthing`.
5. Baseline firewall posture: Hetzner Cloud Firewall at the provider level (allow 22, 22000, 80, 443; deny rest) — UFW alone is misleading when Docker is editing iptables directly.

---

## 6. Onboarding flow (new Linux client joins)

1. **Pre-flight:** confirm invariants from §2.
2. **Wait for Device ID** from Linux-side Claude (see `local-claude.md` step 2).
3. **Register the Linux device** with `autoAcceptFolders: false`:
   ```bash
   curl -sX PUT -H "X-API-Key: $API" -H "Content-Type: application/json" \
     -d '{"deviceID":"<LINUX-ID>","name":"Linux","compression":"metadata","autoAcceptFolders":false,"addresses":["dynamic"]}' \
     "$BASE/config/devices/<LINUX-ID>"
   ```
4. **Add the device to the folder's share list** (append, don't wipe):
   ```bash
   curl -sH "X-API-Key: $API" "$BASE/config/folders/obsidian-vault" \
     | jq '.devices += [{"deviceID":"<LINUX-ID>"}]' \
     | curl -sX PUT -H "X-API-Key: $API" -H "Content-Type: application/json" -d @- \
       "$BASE/config/folders/obsidian-vault"
   ```
5. **Confirm connection:** `GET /system/connections` shows `"connected": true` for the Linux device. If not, check TCP 22000 reachability.
6. **Wait for client confirmation**: folder type `receiveonly`, `receiveOnlyTotalItems: 0`. If nonzero, coordinate a revert before testing.
7. **Run isolation test jointly** (§8).

---

## 7. Wire Claude's memory into the vault

1. Move existing memory from `/root/.claude/projects/-root-projects-bdi-senior-developer/memory/` → `/root/obsidian-vault/claude-memory/`.
2. Remove the now-empty original directory.
3. Symlink: `/root/.claude/.../memory → /root/obsidian-vault/claude-memory`.
4. Verify with a test memory write.
5. Add a `MOC.md` index note in `claude-memory/` for graph-view entry.

---

## 8. Isolation test protocol (joint with Linux-side Claude)

1. You: `ls /root/obsidian-vault/` — snapshot file list.
2. Linux side creates `test-from-linux.md`.
3. Wait ~30 seconds.
4. You: re-list + `curl -sH "X-API-Key: $API" "$BASE/db/status?folder=obsidian-vault" | jq '.globalFiles, .localFiles'`. Counts unchanged, new file absent.
5. Linux side reverts. Confirm gone on their side.
6. Final VPS check: nothing changed, no new `.stversions/` entry.

**Pass criteria:** step 4 shows zero propagation.

**Other propagation checks (should keep working):**
- VPS → Linux file create: `touch /root/obsidian-vault/test-from-vps.md` → lands on Linux within seconds.
- VPS → Linux rename: `mv` on VPS → rename (not duplicate) lands on Linux.
- Claude memory write via symlink → lands in `claude-memory/` on Linux.

---

## 9. Day-to-day ops

- Routine writes: write to `/root/obsidian-vault/`; Syncthing propagates.
- `.stignore` changes propagate on next scan.
- Remote UI: `ssh -L 8384:localhost:8384 root@vps`, then `http://127.0.0.1:8384`.

---

## 10. Rollback plan

- Syncthing install broken: `apt remove syncthing`, `rm -rf /root/.local/state/syncthing`.
- Memory symlink broke Claude: `rm` the symlink, `mv` the directory back.

---

## 11. Gotchas & risks (VPS side)

1. **Loopback-only UI + password** — SSH tunnel for remote reconfig. Accept the tradeoff.
2. **Symlinked memory path** — if Claude's runtime does `realpath` or refuses symlinks, fall back to bind mount or file-sync hook.
3. **Initial sync** will include `.obsidian/` subfolders the Linux client creates (themes, plugins). Mostly filtered by `.stignore`; rest is harmless.
4. **UFW vs Docker** — Docker edits iptables directly, bypassing UFW rules. Use provider-level firewall (Hetzner Cloud Firewall) for public traffic filtering.

---

## 12. What NOT to do

- Do not change folder type back to `sendreceive`.
- Do not `DELETE` a client device without coordinating.
- Do not expose `127.0.0.1:8384` to the public internet.

---

## 13. Why Syncthing, not LiveSync

LiveSync needs a headless Obsidian process to see filesystem writes from Claude — fragile and ~600MB RAM. Syncthing is filesystem-native (~30MB RAM, one daemon) and transparent to any writer.
