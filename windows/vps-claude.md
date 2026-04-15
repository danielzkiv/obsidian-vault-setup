# VPS Claude — Windows Client Setup & Operations

> **Paired file:** [`local-claude.md`](./local-claude.md) — runbook for the Claude instance on the Windows machine. Read it first so you know what the other side will do and what to hand back.

Self-contained runbook for the Claude instance on the **VPS** when onboarding or maintaining a Windows read-only mirror. Includes install, configuration, pairing, memory wiring, tests, rollback, gotchas, and historical execution notes.

---

## 1. Architecture & role model

Set up Syncthing on the headless VPS to sync `/root/obsidian-vault/` with a Windows machine running Obsidian Desktop. Claude's memory is symlinked into the vault so every memory write becomes a synced note.

**Role model:** VPS is the **sole writer**. The Windows client is a **read-only mirror**.
- VPS folder type: `sendonly` — create/edit/delete/rename here propagates outward.
- Client folder type: `receiveonly` — Obsidian can open and render the vault, but any local change is flagged as "Local Additions" and reverted on demand; it never reaches the VPS.
- Rationale: Claude on the VPS is the main author. A read-only client prevents case-collision bugs, accidental Obsidian-plugin writes, and two-writer conflict resolution entirely.

---

## 2. VPS-side invariants to maintain

| Setting | Required value | Why |
|---|---|---|
| Folder `obsidian-vault` type | `sendonly` | VPS-only writes propagate out; nothing from clients is accepted |
| `.stignore` content | `.obsidian/workspace*`, `.obsidian/cache*`, `.obsidian/graph.json`, `.obsidian/app.json`, `.trash/` | Propagates to clients; keeps Obsidian per-device UI state from flagging as "Local Additions" |
| Web UI bind | `127.0.0.1:8384` only | Never exposed; reconfigure via SSH tunnel |
| Versioning | `simple`, keep 5 | Safety net for accidental VPS-side deletes |
| New devices | `autoAcceptFolders: false` | Prevents surprise-accept on client-side quirks |

---

## 3. Install Syncthing on the VPS (first-time)

1. **Install from Syncthing's apt repo** (not Ubuntu's, which lags versions):
   - Add `https://syncthing.net/release-key.gpg` to `/etc/apt/keyrings/syncthing-archive-keyring.gpg`.
   - Add `deb [signed-by=/etc/apt/keyrings/syncthing-archive-keyring.gpg] https://apt.syncthing.net/ syncthing stable` to `/etc/apt/sources.list.d/syncthing.list`.
   - `apt update && apt install syncthing`.
   - Pulls v1.30+ (Ubuntu 24.04's default is 1.27, too old for the CLI changes SyncthingWindowsSetup expects on the client side).
2. **Enable systemd service** as root (so it can read/write the vault): `systemctl enable --now syncthing@root.service`.
3. **Verify:** `systemctl is-active syncthing@root` → `active`; `ss -tlnp | grep -E ':8384|:22000'` → both listeners (loopback 8384, all-interfaces 22000).

---

## 4. Configure Syncthing on the VPS

Everything is driven via the REST API at `http://127.0.0.1:8384/rest/...` with `X-API-Key` header. The key is in `/root/.local/state/syncthing/config.xml`. REST is idempotent and survives restarts; avoid hand-editing XML.

```bash
CONFIG=/root/.local/state/syncthing/config.xml
API=$(grep -oP '<apikey>\K[^<]+' "$CONFIG")
BASE=http://127.0.0.1:8384/rest
```

1. **Bind UI to loopback** — `127.0.0.1:8384` (default; confirm).
2. **Set UI user + password** — `PATCH /rest/config/gui` with `{"user":"admin","password":"<plaintext>"}`; Syncthing bcrypts on save. Defense-in-depth — SSH is the primary control.
3. **Delete auto-created `default` folder** — `DELETE /rest/config/folders/default`.
4. **Add the vault as a `sendonly` folder:**
   ```bash
   curl -sX POST -H "X-API-Key: $API" -H "Content-Type: application/json" \
     -d '{"id":"obsidian-vault","label":"Obsidian Vault","path":"/root/obsidian-vault",
          "type":"sendonly","fsWatcherEnabled":true,"fsWatcherDelayS":5,
          "versioning":{"type":"simple","params":{"keep":"5"}},"devices":[]}' \
     "$BASE/config/folders"
   ```
5. **Write `.stignore`** inside `/root/obsidian-vault/`:
   ```
   .obsidian/workspace*
   .obsidian/cache*
   .obsidian/graph.json
   .obsidian/app.json
   .trash/
   ```
   Propagates to clients automatically.
6. No restart needed — REST changes apply live.

---

## 5. Firewall & connectivity

1. **TCP 22000 reachable from the internet** (Syncthing's sync port). UFW inactive on this VPS → 22000 open by default; otherwise `ufw allow 22000/tcp`.
2. **Leave 21027/udp closed** (LAN discovery, irrelevant over the internet — Syncthing uses global discovery + relays).
3. **Confirm global discovery + relays** — `GET /rest/config/options` → `globalAnnounceEnabled: true`, `relaysEnabled: true`.
4. **Print VPS Device ID** for the client to paste: `syncthing --device-id --home=/root/.local/state/syncthing`.
5. **Baseline firewall posture** — UFW inactive on this VPS is a general weakness but we use **Hetzner Cloud Firewall** at the provider level instead (Docker on this box bypasses UFW via direct iptables manipulation, so UFW would be misleading). Recommended rules: allow inbound TCP 22, 22000, 80, 443; deny rest; allow all outbound.

---

## 6. Onboarding flow (new Windows client joins)

1. **Pre-flight:** confirm invariants from §2. Patch drift. Do not proceed otherwise.
2. **Wait for Device ID** from Windows-side Claude (they hand it to you — see `local-claude.md` step 3).
3. **Register the Windows device** with `autoAcceptFolders: false` explicitly:
   ```bash
   curl -sX PUT -H "X-API-Key: $API" -H "Content-Type: application/json" \
     -d '{"deviceID":"<WINDOWS-ID>","name":"Windows","compression":"metadata","autoAcceptFolders":false,"addresses":["dynamic"]}' \
     "$BASE/config/devices/<WINDOWS-ID>"
   ```
4. **Add the device to the folder's share list** (fetch current, append, PUT back — don't wipe):
   ```bash
   curl -sH "X-API-Key: $API" "$BASE/config/folders/obsidian-vault" \
     | jq '.devices += [{"deviceID":"<WINDOWS-ID>"}]' \
     | curl -sX PUT -H "X-API-Key: $API" -H "Content-Type: application/json" -d @- \
       "$BASE/config/folders/obsidian-vault"
   ```
5. **Confirm connection** — `GET /rest/system/connections` shows `"connected": true` for the Windows device. If not, check 22000 reachability from client side.
6. **Wait for client confirmation** that folder type is `receiveonly` and `receiveOnlyTotalItems: 0`. If nonzero, coordinate a Revert Local Changes before proceeding.
7. **Run isolation test jointly** (§8).

---

## 7. Wire Claude's memory into the vault

1. **Move existing memory** from `/root/.claude/projects/-root-projects-bdi-senior-developer/memory/` → `/root/obsidian-vault/claude-memory/`. If empty, just create the target.
2. **Remove the now-empty original directory.**
3. **Create a symlink:**
   ```
   /root/.claude/projects/-root-projects-bdi-senior-developer/memory
     → /root/obsidian-vault/claude-memory
   ```
4. **Verify** — write a test memory, confirm it lands in the vault.
5. **Add a `MOC.md`** (Map of Content) that indexes `MEMORY.md` so it's a graph-view entry point in Obsidian.

---

## 8. Isolation test protocol (joint with Windows-side Claude)

Verifies the read-only contract in actual behavior, not just config.

1. Your turn: `ls /root/obsidian-vault/` — snapshot the current file list; report it.
2. Windows side creates `test-from-windows.md`.
3. Wait ~30 seconds.
4. Your side: re-list `/root/obsidian-vault/` — the new file must **not** appear. Also:
   ```bash
   curl -sH "X-API-Key: $API" "$BASE/db/status?folder=obsidian-vault" | jq '.globalFiles, .localFiles'
   ```
   Counts unchanged from pre-test.
5. Windows side clicks **Revert Local Changes**. Confirm file is gone on their machine.
6. Final check on VPS: nothing changed; nothing in `.stversions/` from the test.

**Pass criteria:** step 4 shows zero propagation. Any propagation = folder type drifted or was never applied. Fix before marking setup done.

**Other propagation tests (should keep working):**
- VPS → Windows file create: `touch /root/obsidian-vault/test-from-vps.md` → lands on Windows within seconds.
- VPS → Windows rename: `mv` a file on VPS → rename (not duplicate) lands on Windows.
- Claude memory write via symlink → lands in `claude-memory/` on Windows.

---

## 9. Day-to-day ops

- **Routine writes:** just write to `/root/obsidian-vault/`; Syncthing handles propagation.
- **`.stignore` changes:** edit the file; next scan pushes the new patterns to clients.
- **Remote UI access:** SSH tunnel (`ssh -L 8384:localhost:8384 root@vps`), then browse `http://127.0.0.1:8384`.

---

## 10. Rollback plan

- **Syncthing install broke something:** `apt remove syncthing`, `rm -rf /root/.local/state/syncthing`, back to clean slate.
- **Memory symlink breaks Claude:** `rm` the symlink, `mv` the directory back — Claude's memory system is untouched up to that point.

---

## 11. Gotchas & risks (VPS side)

1. **Loopback-only UI + password** — to reconfigure Syncthing remotely, open an SSH tunnel. Accept the tradeoff vs exposing the UI.
2. **Symlinked memory path** — if Claude's runtime ever does a `realpath` check or refuses symlinks, fall back to a bind mount or file-sync hook. Not expected but possible.
3. **Initial sync includes `.obsidian/` subfolders** Windows Obsidian creates (themes, plugins) — these sync back to VPS. Harmless but noted.
4. **UFW vs Docker** — UFW can't cleanly filter traffic when Docker is publishing ports (Docker edits iptables directly, bypassing UFW). Use provider-level firewall (Hetzner Cloud Firewall) instead.

---

## 12. What NOT to do

- Do not change folder type back to `sendreceive` for convenience. If a client needs write access, use a separate repo/vault.
- Do not `DELETE` a client device to "reset" it without coordinating — the initial resync is not free.
- Do not expose `127.0.0.1:8384` to the public internet.

---

## 13. Historical execution notes (2026-04-15 run)

Preserved so the runbook matches what actually happened.

**VPS side setup:**
- Syncthing v1.30.0 installed from upstream apt repo; `syncthing@root.service` running.
- UI bound to `127.0.0.1:8384`, user `admin` + generated password set.
- Folder `obsidian-vault` at `/root/obsidian-vault`, Simple versioning (keep 5).
- `.stignore` per spec; default `~/Sync` folder deleted.
- VPS Device ID: `BVVAFG3-NQXBGAN-RRWY65A-EN63EDU-W7M6JVL-ZEUMN76-DTYE4RE-MRQVMA4`.
- UFW inactive; 22000/tcp reachable externally.

**Pairing:**
- Windows Device ID `SC4OMBT-DXFA2MJ-5GYZZF3-352NVIF-YJNCWUK-LUBDVUO-WZAI5Z4-VD3EYAB` added as trusted device (name `Windows`, compression: metadata).
- Folder shared to Windows; connection established to `95.135.208.15:443` within seconds.

**Role-model correction (post-initial-setup):**
- Folder patched from `sendreceive` → `sendonly` via REST.
- `.stignore` updated to expanded pattern set so Obsidian UI state doesn't flag as "Local Additions."
- Windows side auto-accepted the offer as `sendreceive` (installer template defaulted `autoAcceptFolders=true`) then patched to `receiveonly`. Post-correction: `receiveOnlyTotalItems: 0`, all 47 files synced, no conflicts. `autoAcceptFolders=false` set on the VPS device entry on Windows to prevent recurrence.

**Phase results:**
| Test | Result |
|---|---|
| VPS → Windows | ✅ |
| Windows → VPS (sendreceive era) | ✅ (superseded by isolation test) |
| Rename propagation | ✅ |
| Claude memory via symlink | ✅ all 3 files reached Windows |

**Isolation re-test (still pending — needs to be run):**
| Test | Expected | Run? |
|---|---|---|
| Windows creates file → must NOT appear on VPS | Local Addition on Windows; VPS `ls` unchanged | ⬜ |
| Revert Local Changes on Windows | Local file removed; VPS unchanged | ⬜ |

---

## 14. Why Syncthing, not LiveSync

LiveSync is an Obsidian plugin syncing via CouchDB through Obsidian's internal file-change events — nothing watches the filesystem. Claude writing directly to the vault would be invisible unless a headless Obsidian also ran on the VPS (fragile, ~600MB RAM, Electron-in-Xvfb). Syncthing is filesystem-native (~30MB RAM, one daemon).

| | LiveSync | Syncthing |
|---|---|---|
| Works with FS-level writes (Claude) | no | yes |
| RAM on VPS | ~600MB | ~30MB |
| Moving parts | 3 | 1 |
| Setup complexity | High | Low |
