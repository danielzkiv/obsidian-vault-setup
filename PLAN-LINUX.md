# Plan: Syncthing + Claude Memory Integration (Linux client)

## Overview
Same architecture as the Windows variant but with a **Linux desktop** as the client. Syncthing on the VPS ↔ Syncthing on the Linux desktop ↔ Obsidian Desktop opens the locally-synced folder. Claude's memory symlinks into the vault.

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

1. Bind web UI to loopback only (`127.0.0.1:8384`)
2. Set a web-UI password
3. Add folder `obsidian-vault` at `/root/obsidian-vault`, type Send & Receive, file-watcher on, Simple versioning (5 versions)
4. Write `.stignore`:
   ```
   .obsidian/workspace.json
   .obsidian/workspace-*.json
   .obsidian/cache
   .trash/
   ```
5. Restart Syncthing

---

## Phase 3 — Firewall + connectivity
*(Identical to PLAN-WINDOWS.md Phase 3)*

1. Open TCP 22000 on UFW
2. Leave 21027/udp closed
3. Confirm global discovery + relays enabled
4. Print VPS Device ID for pairing

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

Navigate to `http://localhost:8384` — your **Device ID** is under *Actions → Show ID* (top right).

### 4.5 Pair with the VPS

1. Web UI → *Add Remote Device* → paste the VPS Device ID
2. **Tell me when done** — I'll approve the reverse prompt on the VPS side (or pre-add your Device ID)
3. When the folder share prompt appears, accept `obsidian-vault` and pick a local path, e.g. `~/ObsidianVault`
4. Wait for initial sync (near-instant for a small vault)

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
*(Identical to PLAN-WINDOWS.md Phase 6)*

1. VPS → Linux: create `test-from-vps.md`, confirm it lands in Obsidian
2. Linux → VPS: create a note in Obsidian, confirm it lands on VPS
3. Claude memory: trigger a memory save, confirm it syncs to the Linux client
4. Clean up test files

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
