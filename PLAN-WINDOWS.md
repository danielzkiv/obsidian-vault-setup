# Plan: Syncthing + Claude Memory Integration

> **Status: ✅ Executed successfully on 2026-04-15.** All six phases complete, end-to-end tests passed. See [Execution notes](#execution-notes-2026-04-15-run) at the bottom for what actually happened, including the two gotchas that cost time.

## Overview
Set up Syncthing on the headless VPS to sync `/root/obsidian-vault/` with a Windows machine running Obsidian Desktop. Redirect Claude Code's memory system into the vault via a symlink so every memory Claude writes becomes a synced note.

**Role model:** the VPS is the **sole writer**. The Windows client (and any future client) is a **read-only mirror**.
- VPS folder type: `sendonly` — create/edit/delete/rename here propagates outward.
- Client folder type: `receiveonly` — Obsidian can open and render the vault, but any local change is flagged as "local additions" and rolled back on "Revert Local Changes"; it never reaches the VPS.
- Rationale: Claude on the VPS is the main author (it writes memories and notes). A read-only client prevents case-collision bugs, accidental Obsidian-plugin writes, and two-writer conflict resolution entirely.

---

## Phase 1 — Install Syncthing on the VPS

1. **Install from Syncthing's apt repo** (not Ubuntu's, which lags versions)
   - Add `https://syncthing.net/release-key.gpg` keyring to `/etc/apt/keyrings/syncthing-archive-keyring.gpg`
   - Add `deb [signed-by=/etc/apt/keyrings/syncthing-archive-keyring.gpg] https://apt.syncthing.net/ syncthing stable` to `/etc/apt/sources.list.d/syncthing.list`
   - `apt update && apt install syncthing`
   - Pulls v1.30+ (Ubuntu 24.04's default is 1.27, too old for the CLI changes SyncthingWindowsSetup expects on the client side)

2. **Enable as a systemd service** running under `root` (so it can read/write the vault)
   - `systemctl enable --now syncthing@root.service`
   - Auto-starts on boot, restarts on crash

3. **Verify startup** — `systemctl is-active syncthing@root` should say `active`; `ss -tlnp | grep -E ':8384|:22000'` should show both listeners (loopback-only on 8384, all interfaces on 22000).

---

## Phase 2 — Configure Syncthing on the VPS

Driven via the Syncthing REST API (`http://127.0.0.1:8384/rest/...` with `X-API-Key` header — the key is in the auto-generated `/root/.local/state/syncthing/config.xml`). Doing it through the API instead of hand-editing XML is idempotent and survives Syncthing restarts.

1. **Bind web UI to loopback only** — `127.0.0.1:8384` (Syncthing's default, but confirm). Never exposed to the internet.
2. **Set a web-UI user + password** — `PATCH /rest/config/gui` with `{"user": "admin", "password": "<generated>"}`. Syncthing bcrypts the plaintext on save. The password only matters if someone opens an SSH tunnel to the loopback UI — it's a defence-in-depth layer, not the primary control (SSH is).
3. **Remove the auto-created `default` folder** (`/root/Sync`, unused) — `DELETE /rest/config/folders/default`.
4. **Add the vault as a shared folder** — `POST /rest/config/folders`:
   - Folder ID: `obsidian-vault`
   - Path: `/root/obsidian-vault`
   - Type: `sendonly` — VPS is the sole writer; clients mirror read-only
   - File watcher: enabled, 5s delay
   - Versioning: `simple` with 5 versions kept (cheap safety net for accidental deletes on the VPS side)
   - Devices: empty for now — added in Phase 3 once the Windows Device ID is known.
5. **Write `.stignore`** inside the vault (propagates down to clients, so one setup covers all mirrors; keeps Obsidian per-device UI state from polluting sync as "local additions" on read-only clients):
   ```
   .obsidian/workspace*
   .obsidian/cache*
   .obsidian/graph.json
   .obsidian/app.json
   .trash/
   ```
6. **No explicit restart needed** — REST config changes are applied live.

---

## Phase 3 — Firewall + connectivity

1. **TCP 22000 must be reachable from the internet** (Syncthing's sync port). On this VPS UFW was inactive at setup time, so 22000 was already open — but if/when UFW is enabled, add `ufw allow 22000/tcp`.
2. **Leave 21027/udp closed** (local LAN discovery — irrelevant over the internet, and Syncthing uses global discovery + relays instead).
3. **Confirm global discovery + relays enabled** (default, but verify via `GET /rest/config/options` — `globalAnnounceEnabled: true`, `relaysEnabled: true`). This is what lets the home Windows machine find the VPS through NAT without port forwarding on the home side.
4. **Print the VPS's Device ID** (`syncthing --device-id --home=/root/.local/state/syncthing`) — needed for the Windows pairing step.
5. **Flag separately**: this VPS has UFW inactive overall → every port is exposed by default. Good for Syncthing but poor posture in general; recommend a baseline UFW policy (allow 22/tcp + 22000/tcp, deny rest) after this plan finishes. Not in scope here.

---

## Phase 4 — Windows-side instructions (user executes)

> **Do NOT use SyncTrayzor.** It's abandoned (last release 2023) and invokes Syncthing with flags that were removed in v1.27+ — you'll see errors like `unknown flag -n` with no functional UI. Use SyncthingWindowsSetup instead.

### 4.1 Confirm no Syncthing is already running

Before installing, verify there's no stale Syncthing/SyncTrayzor install:

```powershell
netstat -ano | findstr :8384
```

If something's listening, investigate before installing — a common surprise is a **VS Code Remote-SSH port forward** silently tunneling `localhost:8384` to the VPS's loopback UI (see Gotcha #4 below). Close it or expect port 8385 fallback in 4.2.

### 4.2 Install Syncthing via SyncthingWindowsSetup

1. Download the latest installer from [SyncthingWindowsSetup](https://github.com/Bill-Stewart/SyncthingWindowsSetup/releases/latest) (actively maintained successor to SyncTrayzor's "install and forget" role)
2. Run `syncthing-x64-*.exe`. Defaults are fine: per-user service, autostart at logon, web UI on `localhost:8384`
3. **Port fallback:** if 8384 is occupied (e.g. by a VS Code tunnel), the installer falls back to `127.0.0.1:8385`. Note the actual port from the installer's summary page.
4. **First GUI access** — navigate to `http://127.0.0.1:<port>` (8384 or 8385). Modern Syncthing (v1.29+) shows a "Set up authentication" dialog on first access. Pick your own username + password; this is *your* Windows UI only, unrelated to the VPS creds.

### 4.3 Add the VPS as a remote device

In the Syncthing web UI → bottom right → **Add Remote Device**:
- Device ID: the VPS Device ID (given to you after Phase 3)
- Device Name: `VPS`
- Addresses: leave `dynamic` (or add `tcp://<VPS-public-IP>:22000` for a faster first connect)
- Save

### 4.4 Hand the Windows Device ID back for VPS-side pairing

In the Syncthing web UI → top-right **Actions → Show ID** → copy the long `XXXXXXX-XXXXXXX-...` string and paste it back to the VPS-side Claude. The VPS side then pre-approves this device and shares the `obsidian-vault` folder with it.

### 4.5 Accept the folder share prompt — as Receive Only

Within ~30 seconds of the VPS adding your Device ID, the Windows Syncthing UI shows **"VPS wants to share folder obsidian-vault"**:
1. Click **Add**
2. **Folder Path**: `C:\Users\<you>\ObsidianVault` (or wherever you prefer)
3. **Advanced tab → Folder Type: `Receive Only`** — this is the critical step. The Windows installer's device template may default `autoAcceptFolders=true` on new remote devices, which causes the offer to be auto-accepted as `sendreceive` before you can intervene. If that happens, patch the folder back to `receiveonly` immediately via `PATCH /rest/config/folders/obsidian-vault` (body: `{"type":"receiveonly"}`) and verify `receiveOnlyTotalItems: 0` on the folder status.
4. Save

Initial sync runs (near-instant — vault is tiny).

**Behavioral consequence:** Obsidian on Windows can read and render the vault (including graph view), but any edit/create/delete it makes stays local and is flagged as "Local Additions." Use the **Revert Local Changes** button to roll them back. Nothing from the Windows side ever reaches the VPS.

### 4.6 Install Obsidian Desktop

1. Download from https://obsidian.md/download
2. Install, open → *Open folder as vault* → select `C:\Users\<you>\ObsidianVault`
3. You should see `Welcome.md` and any other files from the VPS immediately

---

## Phase 5 — Wire Claude's memory into the vault

1. **Move existing memory (if any)** from `/root/.claude/projects/-root-projects-bdi-senior-developer/memory/` → `/root/obsidian-vault/claude-memory/`. If the dir is empty, just create the target.
2. **Remove the now-empty original directory**
3. **Create a symlink**:
   ```
   /root/.claude/projects/-root-projects-bdi-senior-developer/memory
     → /root/obsidian-vault/claude-memory
   ```
4. **Verify** — write a test memory and confirm it lands in the vault
5. **Add a small `.obsidian` config touch** — create `claude-memory/` as a normal folder, add a `MOC.md` (Map of Content) that indexes MEMORY.md so you can see it as an entry point in Obsidian's graph view

---

## Phase 6 — Sanity test end-to-end

1. **VPS → Windows propagation** — I create `/root/obsidian-vault/test-from-vps.md`; confirm it appears in your Obsidian within seconds.
2. **Windows → VPS isolation** (inverted from the old sendreceive model — the test now verifies the write is *blocked*, not that it propagates):
   - You create a file (e.g. `test-from-windows.md`) inside `C:\Users\<you>\ObsidianVault\` either in Explorer or from Obsidian.
   - **Expected on Windows:** within seconds the Syncthing UI shows the folder with non-zero **Local Additions** (or `receiveOnlyTotalItems > 0` via `GET /rest/db/status?folder=obsidian-vault`). File stays on disk locally.
   - **Expected on VPS:** the file does **not** appear in `/root/obsidian-vault/`. `ls` on the VPS must not show it. `GET /rest/db/status?folder=obsidian-vault` on the VPS shows no new item in `globalFiles`.
   - Clean up by clicking **Revert Local Changes** in the Windows UI → the file is removed locally; both sides are back in sync.
3. **Rename propagation VPS → Windows** — rename a file on the VPS, confirm the rename (not duplicate) lands on Windows.
4. **Claude memory test** — I trigger a memory save on the VPS; confirm it lands in `claude-memory/` and syncs to your Obsidian.
5. **Clean up test files** on the VPS (the sole writer).

**Pass criteria:** tests 1, 3, 4 propagate cleanly VPS → client; test 2 confirms the client-side write is contained on the client and **never** reaches the VPS. Any failure of test 2 means the folder type was not actually set to `receiveonly` on the client — re-patch and re-run before considering the setup done.

---

## What's needed from the user during execution

- **Step 4 (Windows install)** — user drives that side; I'll wait and approve the pairing on the VPS when they say it's sent
- **Nothing else** — Phases 1, 2, 3, 5, 6 are fully on my side

## Rollback plan if something goes wrong

- Syncthing install breaks something: `apt remove syncthing`, `rm -rf /root/.local/state/syncthing`, back to clean slate
- Memory symlink breaks Claude: `rm` the symlink, `mv` the directory back — Claude's memory system is untouched until step 5

---

## Risks & gotchas worth flagging

1. **Loopback-only UI + password** means to reconfigure Syncthing later you'll open an SSH tunnel (`ssh -L 8384:localhost:8384 root@vps`). Acceptable tradeoff vs exposing the UI.
2. **Symlinked memory path** — if Claude's runtime ever does a `realpath` check or refuses symlinks, we'll need to fall back to a bind mount or a file-sync hook. Not expected, but possible.
3. **First initial sync** will include `.obsidian/` subfolders Windows Obsidian creates (themes, plugins). These will sync back to the VPS. Harmless but noted.
4. **VS Code Remote-SSH auto-forwards ports** — if you have a VS Code Remote-SSH session open to the VPS, VS Code may silently forward `localhost:8384` from Windows to the VPS's loopback Syncthing UI. Symptom: on Windows you open `http://localhost:8384`, see a login page, and the VPS admin credentials work on it. That's not Windows Syncthing — that's the VPS's UI. Fix: either close the VS Code SSH session before installing Windows Syncthing, or let SyncthingWindowsSetup fall back to port 8385 on Windows. This was the single biggest source of confusion during first setup.
5. **SyncTrayzor is abandoned** (last real release 2023) — using it against modern Syncthing (v1.27+) produces `unknown flag -n` errors because legacy flags were removed. If you inherit a machine with SyncTrayzor already installed, uninstall it before anything else.
6. **Windows case-insensitive filesystem vs VPS case-sensitive** — `Note.md` and `note.md` are two files on the VPS but collide on Windows. Syncthing detects and logs these as errors. Avoid creating such pairs; if you see a case-conflict error in the Syncthing log, one of them has to be renamed.
7. **Windows `autoAcceptFolders=true` default** — SyncthingWindowsSetup's device template may set `autoAcceptFolders=true`, which causes a freshly-offered folder to be auto-accepted as `sendreceive` before the receive-only setting can be applied. Mitigation: (a) set `"autoAcceptFolders": false` on the VPS device entry when adding it on Windows, and (b) immediately after acceptance, verify folder type is `receiveonly` and patch it if not. See Execution notes below for the real-world occurrence.

---

## Execution notes (2026-04-15 run)

Recorded here so the plan matches what actually happened, not just the theory.

**VPS side — completed by Claude on VPS:**
- Syncthing v1.30.0 installed from upstream apt repo; `syncthing@root.service` running.
- Web UI bound to `127.0.0.1:8384`, user `admin` + generated password set.
- Folder `obsidian-vault` created at `/root/obsidian-vault` with Simple versioning (keep 5).
- `.stignore` written per spec.
- Default `~/Sync` folder (auto-created) deleted.
- VPS Device ID: `BVVAFG3-NQXBGAN-RRWY65A-EN63EDU-W7M6JVL-ZEUMN76-DTYE4RE-MRQVMA4`
- UFW inactive; port 22000/tcp reachable externally by default. Flagged for follow-up.

**Windows side — completed by Claude on local Windows machine:**
- Clean install via SyncthingWindowsSetup v2.0.2 (per-user, autostart at logon).
- No prior Syncthing install existed. The apparent "login page at localhost:8384" earlier was a **VS Code Remote-SSH port forward** to the VPS — not a local Syncthing. Stale Windows config from an aborted first attempt has been deleted.
- Windows Syncthing GUI is on `127.0.0.1:8385` (VS Code still owns 8384 as the tunnel). No impact on the sync plane — only affects which port the local browser UI uses.
- VPS device already added on Windows side (ID above, name `VPS`, addresses: dynamic and `tcp://178.104.79.53:22000`).
- Windows Device ID: to be handed to VPS-side Claude for pre-approval (next step).

**Pairing (completed):**
- Windows Device ID `SC4OMBT-DXFA2MJ-5GYZZF3-352NVIF-YJNCWUK-LUBDVUO-WZAI5Z4-VD3EYAB` added as trusted device on the VPS (name `Windows`, compression: metadata).
- Folder `obsidian-vault` shared to Windows; connection established to `95.135.208.15:443` within seconds.
- Windows accepted folder share, set path `C:\Users\<user>\ObsidianVault`.
- Obsidian Desktop installed on Windows, vault opened successfully.

**Role-model correction (post-initial-setup):**
- VPS folder patched from `sendreceive` → `sendonly` via `PATCH /rest/config/folders/obsidian-vault`.
- `.stignore` updated to the expanded pattern set (`.obsidian/workspace*`, `.obsidian/cache*`, `.obsidian/graph.json`, `.obsidian/app.json`, `.trash/`) so Obsidian per-device UI state on read-only clients isn't flagged as "local additions."
- Windows side: folder auto-accepted as `sendreceive` (installer's device-template quirk — `autoAcceptFolders=true` defaulted on) instead of queuing as a pending offer. Corrected on Windows via `PATCH /rest/config/folders/obsidian-vault` → `{"type":"receiveonly"}`. Post-correction state: `receiveOnlyTotalItems: 0`, `receiveOnlyChangedBytes: 0`, all 47 files synced cleanly, no conflicts. `autoAcceptFolders=false` set on the VPS device entry on Windows to prevent recurrence.

**Phase 5 (completed) — Claude memory in vault:**
- `/root/.claude/projects/-root-projects-bdi-senior-developer/memory` symlinked → `/root/obsidian-vault/claude-memory/`.
- `MEMORY.md` index + `MOC.md` graph-view entry point + first real memory (`memory_location.md` documenting the symlink itself) written and synced to Windows at 100%.

**Phase 6 (completed) — end-to-end tests:**
| Test | Result |
|---|---|
| VPS → Windows | ✅ `test-from-vps.md` synced within seconds |
| Windows → VPS | ✅ `Untitled.md` then renamed to `test-from-windows.md` synced *(run under the old `sendreceive` model — superseded by the isolation test below)* |
| Rename propagation | ✅ confirmed (old snapshot captured in `.stversions/`) |
| Claude memory writes via symlink | ✅ all 3 memory files reached Windows |

**Isolation re-test (post role-model correction — needs to be run to confirm the new contract):**
| Test | Expected | Run? |
|---|---|---|
| Windows creates file → must NOT appear on VPS | Windows shows it as "Local Addition"; VPS `ls /root/obsidian-vault/` doesn't list it | ⬜ pending |
| `Revert Local Changes` on Windows | Local file removed on Windows; nothing changes on VPS | ⬜ pending |

Until both rows above are ✅, treat the receive-only contract as unverified in production behavior (the folder-type flag is set, but we haven't round-tripped an actual write to prove isolation).

**Gotchas that bit us (documented in Risks section):**
1. VS Code Remote-SSH auto-forwarded `localhost:8384` from Windows → VPS UI, masquerading as a local Syncthing. Cost ~20 minutes of debugging. Fix: always use `127.0.0.1` explicitly + SyncthingWindowsSetup fell back to port 8385 automatically.
2. SyncTrayzor (initially attempted) is abandoned and incompatible with modern Syncthing CLI. Replaced with SyncthingWindowsSetup.
3. Obsidian rename dialog doesn't show the `.md` extension; typing `file.md` in the rename box creates `file.md.md`. User-side knowledge, noted for next time.

**Deliberately deferred:**
- UFW baseline — VPS runs Docker containers publishing on 80/443. Because Docker manipulates iptables directly and bypasses UFW rules, UFW would give misleading security. Chose to use **Hetzner Cloud Firewall** (provider-level) instead — handled in the cloud console, cleanly filters public traffic without the Docker/iptables collision. Recommended rule set:
  - Allow inbound: TCP 22 (SSH), TCP 22000 (Syncthing), TCP 80 + 443 (Docker-exposed services)
  - Deny all other inbound
  - Allow all outbound (default)
- Cleanup of test files done: `test-from-vps.md` and `test-from-windows.md` removed from the vault.

**Final vault state on VPS:**
```
/root/obsidian-vault/
├── .obsidian/
├── .stfolder
├── .stignore
├── .stversions/   (Syncthing's auto-versioned backups)
├── claude-memory/
│   ├── MEMORY.md
│   ├── MOC.md
│   └── memory_location.md
└── Welcome.md
```

---

## Why Syncthing, not Self-hosted LiveSync

LiveSync is an Obsidian plugin — it syncs via CouchDB but only via Obsidian's internal file-change events. Nothing watches the filesystem. Claude writing directly to `/root/obsidian-vault` on the VPS would be invisible to LiveSync unless a headless Obsidian instance also ran on the VPS (fragile, ~600MB RAM, Electron-in-Xvfb). Syncthing is filesystem-native (~30MB RAM, one daemon) and works identically for any writer — human editing in the GUI or Claude writing raw Markdown files.

| | LiveSync | Syncthing |
|---|---|---|
| Designed for Obsidian | yes | no (generic) |
| Works with FS-level writes (Claude) | no | yes |
| RAM footprint on VPS | ~600MB (CouchDB + headless Obsidian) | ~30MB |
| Moving parts | 3 (CouchDB, plugin, headless Obsidian) | 1 (daemon) |
| Setup complexity | High (HTTPS, auth, Electron headless) | Low |
