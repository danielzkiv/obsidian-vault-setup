# Linux-Local Claude — Read-Only Mirror Setup & Operations

> **Paired file:** [`vps-claude.md`](./vps-claude.md) — runbook for the Claude instance on the VPS. Read it first so you know what the other side will do and what to hand back.

Self-contained runbook for the Claude instance on a **Linux desktop/laptop** client. Covers install, pairing, receive-only acceptance, Obsidian setup, tests, gotchas.

---

## 1. Architecture & role model

Your machine syncs `~/ObsidianVault` (read-only mirror) against the VPS's `/root/obsidian-vault/` (sole writer). Obsidian Desktop opens the local folder.

- VPS folder type: `sendonly`.
- Your folder type: `receiveonly` — Obsidian can read/render everything; local edits are flagged as "Local Additions" and reverted on demand. They never reach the VPS.
- Rationale: prevents two-writer conflicts and accidental plugin writes on the client side.

---

## 2. Linux-side invariants to achieve

| Setting | Required value | Why |
|---|---|---|
| Syncthing service | `systemctl --user` (not root) | Correct ownership on synced files |
| Lingering | `loginctl enable-linger "$USER"` | Keeps service running after logout |
| inotify watches | `fs.inotify.max_user_watches=524288` | Obsidian + Syncthing blow past 8192 default |
| VPS device entry | `autoAcceptFolders: false` | Prevents surprise-accept as `sendreceive` |
| Folder `obsidian-vault` type | `receiveonly` | Local edits contained; VPS state authoritative |
| `receiveOnlyTotalItems` | `0` | Proves no stray local divergence |
| Folder path | `~/ObsidianVault` | VPS doesn't care about the path, only folder ID |

---

## 3. Install Syncthing

Pick per distro:
- **Debian/Ubuntu:** add Syncthing's apt repo, `sudo apt install syncthing`.
- **Fedora/RHEL:** `sudo dnf install syncthing`.
- **Arch/Manjaro:** `sudo pacman -S syncthing`.
- **openSUSE:** `sudo zypper install syncthing`.

Enable as user systemd service (not root):

```bash
systemctl --user enable --now syncthing.service
loginctl enable-linger "$USER"
```

Config lives at `~/.local/state/syncthing/` (or `~/.config/syncthing/` on older installs).

---

## 4. Bump inotify watch limit

Obsidian + Syncthing can blow past the default 8192 watches:

```bash
echo 'fs.inotify.max_user_watches=524288' | sudo tee /etc/sysctl.d/99-syncthing.conf
sudo sysctl --system
```

---

## 5. Find your API key, port, and Device ID

```bash
CONFIG="${XDG_STATE_HOME:-$HOME/.local/state}/syncthing/config.xml"
[ -f "$CONFIG" ] || CONFIG="$HOME/.config/syncthing/config.xml"

API=$(grep -oP '<apikey>\K[^<]+' "$CONFIG")
PORT=$(grep -oP 'address>127\.0\.0\.1:\K\d+' "$CONFIG" | head -1)
BASE="http://127.0.0.1:$PORT/rest"

# Own Device ID
curl -sH "X-API-Key: $API" "$BASE/system/status" | jq -r .myID
```

> **Gotcha:** if `http://localhost:8384` shows a login page that accepts the **VPS's** admin creds, that's a VS Code Remote-SSH port-forward tunneling to the VPS's UI — not your local Syncthing. Use `http://127.0.0.1:$PORT` explicitly or close the VS Code SSH session.

---

## 6. Onboarding flow

1. **Pre-flight:** `systemctl --user is-active syncthing` → `active`; lingering on (`loginctl show-user "$USER" | grep Linger=yes`); inotify bumped.
2. **Hand your Device ID to the VPS-side Claude.** They'll pre-approve you and share the folder (see `vps-claude.md` §6).
3. **Add the VPS as a remote device** with `autoAcceptFolders: false`:
   ```bash
   curl -sX PUT -H "X-API-Key: $API" -H "Content-Type: application/json" \
     -d '{"deviceID":"<VPS-DEVICE-ID>","name":"VPS","compression":"metadata","autoAcceptFolders":false,"addresses":["dynamic","tcp://<VPS-IP>:22000"]}' \
     "$BASE/config/devices/<VPS-DEVICE-ID>"
   ```
4. **Wait for the folder offer.** Linux Syncthing respects `autoAcceptFolders=false` cleanly (unlike the Windows installer's template quirk), so the offer will queue as pending.
5. **Accept as Receive Only** via the UI (Advanced tab → Folder Type: `Receive Only`, path `~/ObsidianVault`), or via API:
   ```bash
   MYID=$(curl -sH "X-API-Key: $API" "$BASE/system/status" | jq -r .myID)
   curl -sX POST -H "X-API-Key: $API" -H "Content-Type: application/json" \
     -d '{"id":"obsidian-vault","label":"Obsidian Vault","path":"'"$HOME"'/ObsidianVault",
          "type":"receiveonly",
          "devices":[{"deviceID":"'"$MYID"'"},{"deviceID":"<VPS-DEVICE-ID>"}],
          "fsWatcherEnabled":true,"fsWatcherDelayS":10}' \
     "$BASE/config/folders"
   ```
6. **Verify clean state:**
   ```bash
   curl -sH "X-API-Key: $API" "$BASE/config/folders/obsidian-vault" | jq .type
   # "receiveonly"
   curl -sH "X-API-Key: $API" "$BASE/db/status?folder=obsidian-vault" | jq '{receiveOnlyTotalItems, receiveOnlyChangedBytes}'
   # both 0
   ```
   If `receiveOnlyTotalItems > 0`, revert:
   ```bash
   curl -sX POST -H "X-API-Key: $API" "$BASE/db/revert?folder=obsidian-vault"
   ```
7. **Report to VPS Claude:** folder type, `receiveOnlyTotalItems`, connection status.
8. **Run isolation test jointly** (§8).

---

## 7. Install Obsidian Desktop

| Method | Command | Notes |
|---|---|---|
| **Flatpak** (recommended) | `flatpak install flathub md.obsidian.Obsidian` | Sandboxed, auto-updates |
| **AppImage** | Download from [obsidian.md](https://obsidian.md), chmod +x, run | Manual updates |
| **.deb** (Debian/Ubuntu) | `sudo dpkg -i obsidian-*.deb` | Official download |
| **Snap** | `sudo snap install obsidian --classic` | Works; Flatpak is better maintained |
| **AUR** (Arch) | `yay -S obsidian` or `sudo pacman -S obsidian` | `extra/obsidian` is in official repos |

Open Obsidian → *Open folder as vault* → select `~/ObsidianVault`.

**Flatpak sandboxing:** Flatpak Obsidian may not see `~/ObsidianVault` by default. Grant access:
```bash
flatpak override --user --filesystem=~/ObsidianVault md.obsidian.Obsidian
```

---

## 8. Isolation test protocol (joint with VPS-side Claude)

1. VPS snapshots `/root/obsidian-vault/`.
2. Your turn: `echo test > ~/ObsidianVault/test-from-linux.md`.
3. Wait ~30 seconds:
   ```bash
   curl -sH "X-API-Key: $API" "$BASE/db/status?folder=obsidian-vault" | jq .receiveOnlyTotalItems
   # > 0
   ```
4. VPS Claude confirms the file did **not** appear on `/root/obsidian-vault/`. If it did, the test failed — folder type is wrong; stop and debug.
5. Revert: `curl -sX POST -H "X-API-Key: $API" "$BASE/db/revert?folder=obsidian-vault"`. File disappears locally.
6. Re-verify `receiveOnlyTotalItems: 0`.

**Pass criteria:** step 4 shows zero propagation.

---

## 9. Day-to-day expectations for the user

- **Obsidian opens and renders normally** — graph view, plugins, all read-side features work.
- **Edits / new notes / deletes stay on Linux only** and are rolled back by Revert Local Changes. Intended contract.
- **Ignore patterns from the VPS's `.stignore`** apply locally — Obsidian UI state doesn't flag as Local Additions under normal use.

---

## 10. Rollback

- Stop + disable: `systemctl --user disable --now syncthing`.
- Purge config + DB: `rm -rf ~/.local/state/syncthing` (or `~/.config/syncthing`).
- Local vault folder (`~/ObsidianVault`) is yours to keep or delete.

---

## 11. Gotchas (Linux side)

1. **inotify watches** — if logs show `too many open files` or `no space left on device` referencing inotify, raise `fs.inotify.max_user_watches` to 1048576.
2. **Flatpak sandboxing** — see §7 for the `flatpak override` to grant vault access.
3. **Systemd user service lingering** — without `loginctl enable-linger`, Syncthing stops when your user logs out.
4. **SELinux / AppArmor** — on enforcing SELinux (Fedora/RHEL), if the vault is on NFS or a non-standard path, you may need extra booleans. Not needed for a local home dir.
5. **VS Code Remote-SSH port-forward** — auto-forwards `localhost:8384` to the VPS's UI. Symptom: login page at `http://localhost:8384` accepts the VPS's creds. Fix: use `http://127.0.0.1:$PORT` explicitly or close the VS Code SSH session.
6. **Multiple Linux devices** — you can pair a laptop + desktop; the VPS stays the sole writer, both clients mirror.

---

## 12. Advantages over a Windows client

- No antivirus interference with file locks.
- inotify beats Windows's `ReadDirectoryChangesW` on high-frequency changes.
- Case-sensitive filenames match the VPS — zero cross-OS filename collisions.
- Same Syncthing binary, config format, and REST API as the VPS — copy templates verbatim.
- Scriptable end-to-end: pairing, folder-add, etc. can be fully automated via REST without clicking through the UI.

---

## 13. What NOT to do

- Do not flip folder type back to `sendreceive`.
- Do not `DELETE` the folder to "reset" without coordinating with the VPS-side Claude.
- Do not expose the Syncthing web UI off-host.
