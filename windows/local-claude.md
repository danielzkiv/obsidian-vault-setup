# Windows-Local Claude — Read-Only Mirror Setup & Operations

> **Paired file:** [`vps-claude.md`](./vps-claude.md) — runbook for the Claude instance on the VPS. Read it first so you know what the other side will do and what to hand back.

Self-contained runbook for the Claude instance on the **Windows machine**. Covers install, pairing, receive-only acceptance, Obsidian setup, tests, gotchas.

---

## 1. Architecture & role model

Your machine syncs `C:\Users\<you>\ObsidianVault` (read-only mirror) against the VPS's `/root/obsidian-vault/` (sole writer). Obsidian Desktop opens the local folder; Claude's memory lands here via the VPS's symlink.

- VPS folder type: `sendonly`.
- Your folder type: `receiveonly` — Obsidian can read/render everything; local edits are flagged as "Local Additions" and reverted on demand. They never reach the VPS.
- Rationale: prevents two-writer conflicts, Windows case-collision bugs, and accidental plugin writes.

---

## 2. Windows-side invariants to achieve

| Setting | Required value | Why |
|---|---|---|
| Syncthing installer | **SyncthingWindowsSetup** (not SyncTrayzor) | SyncTrayzor is abandoned and incompatible with Syncthing v1.27+ |
| VPS device entry | `autoAcceptFolders: false` | Prevents installer's device-template default from auto-accepting folder offers as `sendreceive` |
| Folder `obsidian-vault` type | `receiveonly` | Local edits contained; VPS-originated state is authoritative |
| `receiveOnlyTotalItems` | `0` | Proves no stray local divergence |
| Folder path | `C:\Users\<you>\ObsidianVault` | The VPS side doesn't care about the path, only the folder ID |

---

## 3. Pre-flight checks

Before installing, verify no stale Syncthing/SyncTrayzor install:

```powershell
netstat -ano | findstr :8384
```

If something's listening, investigate. A common surprise is a **VS Code Remote-SSH port-forward** silently tunneling `localhost:8384` to the VPS's loopback UI (see gotchas §11). Close the VS Code SSH session, or expect the installer to fall back to port 8385.

---

## 4. Install Syncthing via SyncthingWindowsSetup

> **Do NOT use SyncTrayzor.** Abandoned (last release 2023), invokes Syncthing with flags removed in v1.27+ (e.g. `-n`). You'll see broken UIs. Use SyncthingWindowsSetup instead.

1. Download the latest installer from [SyncthingWindowsSetup](https://github.com/Bill-Stewart/SyncthingWindowsSetup/releases/latest).
2. Run `syncthing-x64-*.exe`. Defaults are fine: per-user service, autostart at logon, web UI on `localhost:8384`.
3. **Port fallback:** if 8384 is occupied, the installer falls back to `127.0.0.1:8385`. Note the port from the summary page.
4. **First GUI access:** navigate to `http://127.0.0.1:<port>`. v1.29+ shows a "Set up authentication" dialog — pick any username + password; local UI only, unrelated to VPS creds.

---

## 5. Find your API key, port, and Device ID

```powershell
$cfg = "$env:LOCALAPPDATA\Syncthing\config.xml"
$api = (Select-String -Path $cfg -Pattern '<apikey>([^<]+)</apikey>').Matches[0].Groups[1].Value
$port = ((Select-String -Path $cfg -Pattern 'address>127\.0\.0\.1:(\d+)').Matches[0].Groups[1].Value)
$h = @{ "X-API-Key" = $api }
$base = "http://127.0.0.1:$port/rest"

# Own Device ID
(Invoke-RestMethod -Uri "$base/system/status" -Headers $h).myID
```

---

## 6. Onboarding flow

1. **Hand your Device ID to the VPS-side Claude.** They'll pre-approve your device and share the folder (see `vps-claude.md` §6).
2. **Add the VPS as a remote device** with `autoAcceptFolders: false` — critical to prevent the installer template's default from auto-accepting the folder offer as `sendreceive`:
   ```powershell
   $body = @{
     deviceID = "<VPS-DEVICE-ID>"
     name = "VPS"
     compression = "metadata"
     autoAcceptFolders = $false
     addresses = @("dynamic", "tcp://<VPS-IP>:22000")
   } | ConvertTo-Json
   Invoke-RestMethod -Method Put -Uri "$base/config/devices/<VPS-DEVICE-ID>" -Headers $h -ContentType "application/json" -Body $body
   ```
3. **Wait for the folder offer.** Syncthing will either queue it as a pending offer (good) or auto-accept it as `sendreceive` (the installer-template quirk — see §11).
4. **Accept as Receive Only:**
   - If pending: UI → *Add* → set **Advanced tab → Folder Type: `Receive Only`**, path `C:\Users\<you>\ObsidianVault`.
   - If auto-accepted: immediately patch:
     ```powershell
     Invoke-RestMethod -Method Patch -Uri "$base/config/folders/obsidian-vault" -Headers $h -ContentType "application/json" -Body '{"type":"receiveonly"}'
     ```
5. **Verify clean state** — both must hold:
   ```powershell
   (Invoke-RestMethod -Uri "$base/config/folders/obsidian-vault" -Headers $h).type          # receiveonly
   (Invoke-RestMethod -Uri "$base/db/status?folder=obsidian-vault" -Headers $h) |
     Select receiveOnlyTotalItems, receiveOnlyChangedBytes                                   # both 0
   ```
   If `receiveOnlyTotalItems > 0`, click **Revert Local Changes** in the UI or `POST /db/revert?folder=obsidian-vault`.
6. **Report to VPS Claude:** folder type, `receiveOnlyTotalItems`, connection status.
7. **Run isolation test jointly** (§8).

---

## 7. Install Obsidian Desktop

1. Download from https://obsidian.md/download.
2. Install, open → *Open folder as vault* → select `C:\Users\<you>\ObsidianVault`.
3. You should see `Welcome.md` and whatever else is on the VPS within seconds.

---

## 8. Isolation test protocol (joint with VPS-side Claude)

1. VPS-side Claude snapshots `/root/obsidian-vault/`.
2. Your turn: create `C:\Users\<you>\ObsidianVault\test-from-windows.md` with any content (via Explorer or Obsidian).
3. Wait ~30 seconds. Verify:
   ```powershell
   (Invoke-RestMethod -Uri "$base/db/status?folder=obsidian-vault" -Headers $h).receiveOnlyTotalItems
   # > 0
   ```
4. VPS-side Claude confirms the file did **not** appear on `/root/obsidian-vault/`. If it did, the test failed — folder type is wrong; stop and debug.
5. Click **Revert Local Changes** (or `Invoke-RestMethod -Method Post -Uri "$base/db/revert?folder=obsidian-vault" -Headers $h`). Local file disappears.
6. Re-verify `receiveOnlyTotalItems: 0`.

**Pass criteria:** step 4 shows zero propagation from Windows to VPS.

---

## 9. Day-to-day expectations for the user

- **Obsidian opens and renders normally** — graph view, plugins, everything read-side works.
- **Edits / new notes / deletes stay on Windows only** and are rolled back by Revert Local Changes. Intended contract — if they want a real change, coordinate with the VPS side.
- **Ignore patterns from the VPS's `.stignore`** apply locally too — Obsidian UI state (`.obsidian/workspace*` etc.) doesn't show up as Local Additions under normal use, so `receiveOnlyTotalItems` stays at 0.

---

## 10. Rollback

- Uninstall SyncthingWindowsSetup from Programs & Features. Delete `%LOCALAPPDATA%\Syncthing` to remove config + DB. No cleanup needed on VPS side — the stale device entry is harmless (just remove it if desired).

---

## 11. Gotchas (Windows side)

1. **VS Code Remote-SSH auto-forwards ports** — if you have a VS Code SSH session open to the VPS, VS Code silently forwards `localhost:8384` from Windows to the VPS's loopback Syncthing UI. Symptom: `http://localhost:8384` shows a login page that accepts the **VPS's** admin creds. Fix: always use `http://127.0.0.1:<port>` explicitly (VS Code only rewrites `localhost`), or close the VS Code SSH session. Was the single biggest source of confusion during first setup.
2. **SyncTrayzor is abandoned** — if you inherit a machine with it installed, uninstall before anything else.
3. **Windows case-insensitive filesystem vs VPS case-sensitive** — `Note.md` and `note.md` are two files on the VPS but collide on Windows. Syncthing logs this as an error. Under the read-only model the VPS is the only creator, so just avoid creating such pairs there.
4. **`autoAcceptFolders=true` installer-template default** — SyncthingWindowsSetup's device template may set `autoAcceptFolders=true`, causing folder offers to be auto-accepted as `sendreceive` before the receive-only flag can be set. Mitigation: (a) add the VPS device with `autoAcceptFolders: false` explicitly (step 6.2), (b) if auto-accept happened anyway, patch folder type to `receiveonly` immediately (step 6.4 fallback).
5. **Obsidian rename dialog hides `.md`** — typing `file.md` in the rename box creates `file.md.md`. User-side knowledge, no tooling fix.

---

## 12. What NOT to do

- Do not flip folder type back to `sendreceive`. Silent two-writer bug waiting to happen.
- Do not `DELETE` the folder to "reset" without coordinating with the VPS-side Claude — initial resync isn't free.
- Do not expose the Syncthing web UI off-host.
