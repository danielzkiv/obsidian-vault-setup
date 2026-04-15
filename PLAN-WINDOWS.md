# Plan: Syncthing + Claude Memory Integration

## Overview
Set up Syncthing on the headless VPS to sync `/root/obsidian-vault/` with a Windows machine running Obsidian Desktop. Redirect Claude Code's memory system into the vault via a symlink so every memory Claude writes becomes a synced note.

---

## Phase 1 — Install Syncthing on the VPS

1. **Install from Syncthing's apt repo** (not Ubuntu's, which lags versions)
   - Add `https://syncthing.net/release-key.gpg` keyring
   - Add `deb https://apt.syncthing.net/ syncthing stable` to `/etc/apt/sources.list.d/syncthing.list`
   - `apt update && apt install syncthing`
   - Pins us to v1.29+ which has the improved file watcher and relay behavior noted in the research

2. **Enable as a systemd service** running under `root` (so it can read/write the vault)
   - `systemctl enable --now syncthing@root.service`
   - Auto-starts on boot, restarts on crash

3. **Verify startup** — check `systemctl status syncthing@root` and confirm it's listening on `127.0.0.1:8384`

---

## Phase 2 — Configure Syncthing on the VPS

Edit `/root/.local/state/syncthing/config.xml` (or use the web UI via SSH tunnel — I'll do it via the API/config for reproducibility):

1. **Bind web UI to loopback only** — `127.0.0.1:8384` (default, but confirm). Never exposed to the internet.
2. **Set a web-UI password** (so even if the loopback is accessed via tunnel by someone else, it's protected)
3. **Add the vault as a shared folder**:
   - Folder ID: `obsidian-vault`
   - Path: `/root/obsidian-vault`
   - Type: `Send & Receive`
   - File watcher: enabled (default)
   - Versioning: `Simple` with 5 versions (cheap safety net for accidental deletes from either side)
4. **Write `.stignore`** inside the vault:
   ```
   .obsidian/workspace.json
   .obsidian/workspace-*.json
   .obsidian/cache
   .trash/
   ```
5. **Restart Syncthing** to pick up config

---

## Phase 3 — Firewall + connectivity

1. **Open TCP 22000** on UFW (Syncthing's sync port)
2. **Leave 21027/udp closed** (local discovery — irrelevant over the internet, and Syncthing uses the global discovery + relays instead)
3. **Confirm global discovery + relays enabled** (default, but verify) — this is what lets your home Windows machine find the VPS through NAT without port forwarding on your side
4. **Print the VPS's Device ID** so you can paste it into SyncTrayzor

---

## Phase 4 — Windows-side instructions (for you to execute)

Concise handoff:

1. Download + install [SyncTrayzor](https://github.com/canton7/SyncTrayzor) (latest release installer)
2. First-run — it auto-generates your Device ID
3. Add remote device — paste the VPS Device ID; approve the reverse pairing prompt that appears on my side (I'll need you to tell me when you send it so I approve on the VPS)
4. Accept the `obsidian-vault` folder share — pick local path `C:\Users\<you>\ObsidianVault`
5. Wait for initial sync (will be ~instant, vault is nearly empty)
6. Open Obsidian Desktop → "Open folder as vault" → select `C:\Users\<you>\ObsidianVault`

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

1. **VPS → Windows test** — I create a file `/root/obsidian-vault/test-from-vps.md`; confirm it appears in your Obsidian within seconds
2. **Windows → VPS test** — you create a note in Obsidian; I confirm the file appears on VPS
3. **Claude memory test** — I trigger a memory save; confirm it lands in `claude-memory/` and syncs to your Obsidian
4. **Clean up test files**

---

## What's needed from the user during execution

- **Step 4 (Windows install)** — user drives that side; I'll wait and approve the pairing on the VPS when they say it's sent
- **Nothing else** — Phases 1, 2, 3, 5, 6 are fully on my side

## Rollback plan if something goes wrong

- Syncthing install breaks something: `apt remove syncthing`, `rm -rf /root/.local/state/syncthing`, back to clean slate
- Memory symlink breaks Claude: `rm` the symlink, `mv` the directory back — Claude's memory system is untouched until step 5

---

## Risks/considerations worth flagging

1. **Loopback-only UI + password** means to reconfigure Syncthing later you'll open an SSH tunnel (`ssh -L 8384:localhost:8384 root@vps`). Acceptable tradeoff vs exposing the UI.
2. **Symlinked memory path** — if Claude's runtime ever does a `realpath` check or refuses symlinks, we'll need to fall back to a bind mount or a file-sync hook. Not expected, but possible.
3. **First initial sync** will include `.obsidian/` subfolders Windows Obsidian creates (themes, plugins). These will sync back to the VPS. Harmless but noted.

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
