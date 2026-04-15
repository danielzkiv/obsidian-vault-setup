# Linux-Local Claude — Read-Only Mirror Onboarding

> **Paired instruction file:** [`vps-claude.md`](./vps-claude.md) — runbook for the Claude instance on the VPS. Read it first so you know what the other side will be doing and what to hand back.
> **Architecture context:** [`../PLAN-LINUX.md`](../PLAN-LINUX.md) — full setup plan, including the role model.

## Your role

You are running on a **Linux desktop/laptop**. Your job is to make this host a **read-only mirror** of the VPS's `obsidian-vault`. The VPS is the sole writer. Anything you (or Obsidian) create here stays local and gets reverted on next sync pass — that is the intended behavior.

## Linux-side invariants to achieve

| Setting | Required value | Why |
|---|---|---|
| Syncthing service | `systemctl --user` (not root) | Runs as your desktop user; correct ownership on synced files |
| Lingering | `loginctl enable-linger "$USER"` | Service keeps running after logout |
| inotify watches | `fs.inotify.max_user_watches=524288` | Obsidian + Syncthing blow past the 8192 default on real vaults |
| VPS device entry | `autoAcceptFolders: false` | Prevents surprise-accept of folder offers as `sendreceive` |
| Folder `obsidian-vault` type | `receiveonly` | Local edits contained; VPS-originated state is authoritative |
| `receiveOnlyTotalItems` | `0` | Proves no stray local divergence |
| Folder path | `~/ObsidianVault` (or user's preference) | The VPS side does not care about the path, only the folder ID |

## Finding your API key + UI port

```bash
# Config lives in one of these (newer installs use the first)
CONFIG="${XDG_STATE_HOME:-$HOME/.local/state}/syncthing/config.xml"
[ -f "$CONFIG" ] || CONFIG="$HOME/.config/syncthing/config.xml"

API=$(grep -oP '<apikey>\K[^<]+' "$CONFIG")
PORT=$(grep -oP 'address>127\.0\.0\.1:\K\d+' "$CONFIG" | head -1)
BASE="http://127.0.0.1:$PORT/rest"

# Your Device ID
curl -sH "X-API-Key: $API" "$BASE/system/status" | jq -r .myID
```

> **Gotcha:** if you see a login page at `http://localhost:8384` that accepts the **VPS's** admin creds, that's a VS Code Remote-SSH port-forward tunneling to the VPS's UI, not your local Syncthing. Use `http://127.0.0.1:$PORT` explicitly, or close the VS Code SSH session.

## Onboarding flow

1. **Pre-flight:** confirm Syncthing is installed, `systemctl --user is-active syncthing` returns `active`, and lingering is enabled (`loginctl show-user "$USER" | grep Linger=yes`). Bump inotify watches if not already done.
2. **Get your Device ID** (command above) and hand it to the VPS-side Claude.
3. **Add the VPS as a remote device** with `autoAcceptFolders: false`:
   ```bash
   curl -sX PUT -H "X-API-Key: $API" -H "Content-Type: application/json" \
     -d '{"deviceID":"<VPS-DEVICE-ID>","name":"VPS","compression":"metadata","autoAcceptFolders":false,"addresses":["dynamic","tcp://<VPS-IP>:22000"]}' \
     "$BASE/config/devices/<VPS-DEVICE-ID>"
   ```
4. **Wait for the folder offer.** When the VPS shares the folder, Syncthing will queue it as a pending offer (expected — Linux Syncthing respects the `autoAcceptFolders=false` flag cleanly, unlike the Windows installer's template quirk).
5. **Accept as Receive Only via the UI**: Advanced tab → Folder Type: `Receive Only`, path `~/ObsidianVault`. Or via API:
   ```bash
   curl -sX POST -H "X-API-Key: $API" -H "Content-Type: application/json" \
     -d '{"id":"obsidian-vault","label":"Obsidian Vault","path":"'"$HOME"'/ObsidianVault","type":"receiveonly","devices":[{"deviceID":"'"$(curl -sH "X-API-Key: $API" "$BASE/system/status" | jq -r .myID)"'"},{"deviceID":"<VPS-DEVICE-ID>"}],"fsWatcherEnabled":true,"fsWatcherDelayS":10}' \
     "$BASE/config/folders"
   ```
6. **Verify clean state:**
   ```bash
   curl -sH "X-API-Key: $API" "$BASE/config/folders/obsidian-vault" | jq .type
   # -> "receiveonly"
   curl -sH "X-API-Key: $API" "$BASE/db/status?folder=obsidian-vault" | jq '{receiveOnlyTotalItems, receiveOnlyChangedBytes}'
   # -> both 0
   ```
   If `receiveOnlyTotalItems > 0`, revert:
   ```bash
   curl -sX POST -H "X-API-Key: $API" "$BASE/db/revert?folder=obsidian-vault"
   ```
7. **Report back to the VPS-side Claude:** folder type, `receiveOnlyTotalItems`, connection status.
8. **Run the isolation test with them** (see next section).

## Isolation test protocol (run jointly with VPS-side Claude)

1. VPS-side Claude snapshots `/root/obsidian-vault/`.
2. Your turn: `echo "test" > ~/ObsidianVault/test-from-linux.md`.
3. Wait ~30 seconds. Verify:
   ```bash
   curl -sH "X-API-Key: $API" "$BASE/db/status?folder=obsidian-vault" | jq .receiveOnlyTotalItems
   # -> > 0
   ```
4. VPS-side Claude confirms the file did **not** appear on `/root/obsidian-vault/`. If it did, the test failed — folder type is wrong; stop and debug.
5. Revert: `curl -sX POST -H "X-API-Key: $API" "$BASE/db/revert?folder=obsidian-vault"`. The local file disappears.
6. Re-verify `receiveOnlyTotalItems: 0`.

**Pass criteria:** step 4 shows zero propagation from Linux to VPS. Any propagation = folder type drifted or was never applied.

## Day-to-day expectations for the user

- **Obsidian can open the vault and render it** (graph view included). All read-side features work.
- **Edits / new notes / deletes in Obsidian stay on Linux only** and are rolled back on the next Revert Local Changes pass. This is the intended contract.
- **Ignore patterns** from the VPS's `.stignore` apply here too — Obsidian UI state won't show up as Local Additions.

## What NOT to do

- Do not flip folder type back to `sendreceive`.
- Do not `DELETE` the folder to "reset" it without coordinating with the VPS-side Claude.
- Do not expose the Syncthing Web UI off-host.
