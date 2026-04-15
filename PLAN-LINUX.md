# Plan: Syncthing + Claude Memory Integration (Linux client)

## Overview
Same architecture as the Windows variant but with a **Linux desktop** as the client. Syncthing on the VPS ↔ Syncthing on the Linux desktop ↔ Obsidian Desktop opens the locally-synced folder. Claude's memory symlinks into the vault.

**Role model:** the VPS is the **sole writer**. The Linux client (and any other client) is a **read-only mirror**.
- VPS folder type: `sendonly` — create/edit/delete/rename here propagates outward.
- Client folder type: `receiveonly` — Obsidian can open and render the vault, but any local change is flagged as "local additions" and rolled back on "Revert Local Changes"; it never reaches the VPS.
- Rationale: Claude on the VPS is the main author. A read-only client avoids two-writer conflicts and accidental plugin writes.

VPS-side phases (1–3) are **identical** to the Windows plan — only the client side (Phase 4) and a few gotchas differ.

---

## Phase 1 — Install Syncthing on the VPS
*(Identical to PLAN-WINDOWS.md Phase 1)*

1. Add Syncthing's apt repo (`deb https://apt.syncthing.net/ syncthing stable`) + signing key
2. `apt install syncthing`
3. `systemctl enable --now syncthing@root.service`
4. Verify it's listening on `127.0.0.1:8384`

---

## Phase 2 — Configure Syncthing on the VPS
*(Identical to PLAN-WINDOWS.md Phase 2)*

Driven via the Syncthing REST API (`http://127.0.0.1:8384/rest/...` with `X-API-Key` header).

1. Bind web UI to loopback only (`127.0.0.1:8384`)
2. `PATCH /rest/config/gui` → set `admin` user + generated password (bcrypted server-side)
3. `DELETE /rest/config/folders/default` → remove the auto-created unused folder
4. `POST /rest/config/folders` → add folder `obsidian-vault` at `/root/obsidian-vault`, type `sendonly` (VPS is sole writer), file-watcher on (5s delay), `simple` versioning keeping 5
5. Write `.stignore` in `/root/obsidian-vault/` (propagates to clients — keeps Obsidian per-device UI state from flagging as "local additions" on read-only mirrors):
   ```
   .obsidian/workspace*
   .obsidian/cache*
   .obsidian/graph.json
   .obsidian/app.json
   .trash/
   ```
6. No explicit restart needed — REST config changes are applied live.

---

## Phase 3 — Firewall + connectivity
*(Identical to PLAN-WINDOWS.md Phase 3)*

1. TCP 22000 reachable from the internet (add `ufw allow 22000/tcp` if UFW is enabled; on this VPS UFW was inactive, so it was already reachable)
2. Leave 21027/udp closed (local LAN discovery, irrelevant over the internet)
3. Confirm global discovery + relays enabled via `GET /rest/config/options`
4. Print VPS Device ID for pairing (`syncthing --device-id --home=/root/.local/state/syncthing`)
5. Flag separately: UFW inactive overall → recommend baseline policy (allow 22/tcp + 22000/tcp) as follow-up work

---

## Phase 4 — Linux-side setup (user executes)

### 4.1 Install Syncthing

- **Debian / Ubuntu:** add the same Syncthing apt repo used on the VPS, then `sudo apt install syncthing`
- **Fedora / RHEL:** `sudo dnf install syncthing`
- **Arch / Manjaro:** `sudo pacman -S syncthing`
- **openSUSE:** `sudo zypper install syncthing`

### 4.2 Enable as a **user** systemd service (not root — runs as your desktop user)

```bash
systemctl --user enable --now syncthing.service
loginctl enable-linger "$USER"   # keeps it running after logout
```

Config lives at `~/.local/state/syncthing/` (or `~/.config/syncthing/` on older installs).

### 4.3 Bump the inotify watch limit

Obsidian + Syncthing can blow past the default 8192 watches on a vault with many plugins:

```bash
echo 'fs.inotify.max_user_watches=524288' | sudo tee /etc/sysctl.d/99-syncthing.conf
sudo sysctl --system
```

### 4.4 Open the Syncthing web UI

Navigate to `http://127.0.0.1:8384` (use `127.0.0.1` explicitly, not `localhost` — see Gotcha #7 below about tunneled ports).

**First-run:** Syncthing v1.29+ shows a "Set up authentication" dialog. Pick any username + password you'll remember — this is your local UI only, unrelated to the VPS creds.

Your **Device ID** is under *Actions → Show ID* (top right).

### 4.5 Pair with the VPS

1. Web UI → *Add Remote Device* → paste the VPS Device ID, name it `VPS`, leave addresses `dynamic` (or add `tcp://<VPS-public-IP>:22000`). **Uncheck "Auto Accept"** on the Sharing tab — otherwise the folder offer will be auto-accepted as `sendreceive` before you can set it to receive-only.
2. **Hand your Device ID back to the VPS-side Claude** — it pre-approves your device and shares the `obsidian-vault` folder with it
3. Within ~30s a "VPS wants to share folder obsidian-vault" prompt appears on your side → click Add, set folder path to `~/ObsidianVault`, and on the **Advanced tab set Folder Type: `Receive Only`**
4. Initial sync runs (near-instant for a small vault). Verify `receiveOnlyTotalItems: 0` on the folder status — nonzero means something on the client was already written before the receive-only flag landed.

**Behavioral consequence:** Obsidian on Linux can read and render the vault, but any edit/create/delete made locally stays local, is flagged as "Local Additions," and is rolled back by **Revert Local Changes**. Nothing from the client ever reaches the VPS.

### 4.6 Install Obsidian Desktop

Pick one:

| Method | Command | Notes |
|---|---|---|
| **Flatpak** (recommended) | `flatpak install flathub md.obsidian.Obsidian` | Sandboxed, auto-updates |
| **AppImage** | Download from [obsidian.md](https://obsidian.md), chmod +x, run | Manual updates |
| **.deb** (Debian/Ubuntu) | `sudo dpkg -i obsidian-*.deb` | Official download |
| **Snap** | `sudo snap install obsidian --classic` | Works but Flatpak is better maintained |
| **AUR** (Arch) | `yay -S obsidian` or `sudo pacman -S obsidian` | `extra/obsidian` is in official repos |

### 4.7 Open the vault

Obsidian → *Open folder as vault* → select `~/ObsidianVault`.

---

## Phase 5 — Wire Claude's memory into the vault
*(Identical to PLAN-WINDOWS.md Phase 5)*

1. Move existing memory from `/root/.claude/projects/-root-projects-bdi-senior-developer/memory/` into `/root/obsidian-vault/claude-memory/` (or create empty target if none)
2. Remove the original dir
3. Symlink: `/root/.claude/.../memory → /root/obsidian-vault/claude-memory`
4. Verify with a test memory write
5. Add a `MOC.md` index note in `claude-memory/`

---

## Phase 6 — Sanity test end-to-end
*(Shape mirrors PLAN-WINDOWS.md Phase 6 — inverted for the sendonly/receiveonly contract.)*

1. **VPS → Linux propagation** — create `/root/obsidian-vault/test-from-vps.md` on the VPS, confirm it lands in Obsidian on the Linux client.
2. **Linux → VPS isolation** (the inverted test — verifies the write is *blocked*, not that it propagates):
   - Create a file (e.g. `test-from-linux.md`) inside `~/ObsidianVault/` via the shell or Obsidian.
   - **Expected on Linux:** Syncthing UI shows non-zero **Local Additions** (`curl -sH "X-API-Key: $API" "http://127.0.0.1:8384/rest/db/status?folder=obsidian-vault" | jq .receiveOnlyTotalItems` returns > 0). File stays on disk locally.
   - **Expected on VPS:** the file does **not** appear in `/root/obsidian-vault/`. Confirm with `ls` on the VPS and with `GET /rest/db/status?folder=obsidian-vault` — no new item in `globalFiles`.
   - Clean up by clicking **Revert Local Changes** in the Linux UI → file removed locally; both sides back in sync.
3. **Rename propagation VPS → Linux** — rename a file on the VPS, confirm the rename (not duplicate) lands on the Linux client.
4. **Claude memory test** — trigger a memory save on the VPS, confirm it syncs to the Linux client.
5. **Clean up test files** on the VPS (the sole writer).

**Pass criteria:** tests 1, 3, 4 propagate cleanly VPS → client; test 2 confirms the client-side write is contained on the client and never reaches the VPS. Any failure of test 2 means the folder type was not actually set to `receiveonly` on the client — re-patch and re-run before considering the setup done.

---

## Linux-specific gotchas

1. **inotify watches** — if logs show `too many open files` or `no space left on device` referencing inotify, raise `fs.inotify.max_user_watches` to 1048576.
2. **Flatpak sandboxing** — Flatpak Obsidian may not see `~/ObsidianVault` by default. Grant access:
   ```bash
   flatpak override --user --filesystem=~/ObsidianVault md.obsidian.Obsidian
   ```
3. **Systemd user service lingering** — without `loginctl enable-linger`, Syncthing stops when your user logs out.
4. **SELinux / AppArmor** — on enforcing SELinux (Fedora/RHEL), if the vault is on an NFS or non-standard path, you may need extra booleans. Not needed for a local home dir.
5. **Wayland vs X11** — Obsidian runs on both, no action needed. On some Wayland setups, file-open dialogs via portals are slower the first time.
6. **Multiple Linux devices** — you can add a second Linux host (laptop + desktop) by pairing each with the VPS; the VPS stays source of truth, both clients sync from it.
7. **VS Code Remote-SSH auto-forwards ports** — if you have a VS Code Remote-SSH session open to the VPS, VS Code silently forwards `localhost:8384` from your desktop to the VPS's loopback Syncthing UI. Symptom: on your Linux desktop you open `http://localhost:8384`, see a login page, and the VPS admin creds work on it. That's not your local Syncthing — that's the VPS's UI through a tunnel. Fix: always use `http://127.0.0.1:8384` explicitly (VS Code only rewrites `localhost`), or close the VS Code SSH session first. On Linux this is less disruptive than on Windows (systemd user service binds to `127.0.0.1:8384` directly and starts fine alongside the tunnel), but it *will* confuse you the first time.

---

## Linux-specific advantages over the Windows client

1. **No antivirus** interfering with file locks during sync
2. **inotify** beats Windows's `ReadDirectoryChangesW` for high-frequency file changes
3. **Case-sensitive filenames** match the VPS — zero cross-OS filename collisions
4. **Identical Syncthing config semantics** — you can copy `.stignore` and config templates between VPS and desktop verbatim
5. **Scriptable end-to-end** — pairing, folder-add, etc. can be automated via the Syncthing REST API if you want to provision a new Linux client without clicking through the web UI

---

## Why this plan is safer than the Windows variant

- **Case sensitivity parity** — the #1 cross-OS Obsidian sync issue (`Note.md` vs `note.md` collision on case-insensitive Windows) cannot happen Linux↔Linux
- **Ownership/permission preservation** — Syncthing preserves POSIX permissions; Windows has no POSIX concept, so any file created on Windows syncs back with Syncthing's defaults (usually fine, but worth knowing)
- **Unified tooling** — the same `syncthing` binary, same config format, same CLI/API on both ends
