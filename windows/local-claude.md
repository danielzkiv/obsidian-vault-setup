# Windows-Local Claude — Read-Only Mirror Onboarding

> **Paired instruction file:** [`vps-claude.md`](./vps-claude.md) — runbook for the Claude instance on the VPS. Read it first so you know what the other side will be doing and what to hand back.
> **Architecture context:** [`../PLAN-WINDOWS.md`](../PLAN-WINDOWS.md) — full setup plan, including the role model and prior execution notes.

## Your role

You are running on the **Windows machine**. Your job is to make this host a **read-only mirror** of the VPS's `obsidian-vault`. The VPS is the sole writer. Anything you (or Obsidian) create here stays local and gets reverted on next sync pass — that is the intended behavior.

## Windows-side invariants to achieve

| Setting | Required value | Why |
|---|---|---|
| Syncthing installer | SyncthingWindowsSetup (not SyncTrayzor) | SyncTrayzor is abandoned and incompatible with Syncthing v1.27+ |
| VPS device entry | `autoAcceptFolders: false` | Prevents the installer's template default from auto-accepting folder offers as `sendreceive` |
| Folder `obsidian-vault` type | `receiveonly` | Local edits contained; VPS-originated state is authoritative |
| `receiveOnlyTotalItems` | `0` | Proves no stray local divergence |
| Folder path | `C:\Users\<you>\ObsidianVault` (or user's preference) | The VPS side does not care about the path, only the folder ID |

## Finding your API key + UI port

```powershell
# Config location on Windows
$cfg = "$env:LOCALAPPDATA\Syncthing\config.xml"
# Extract API key
Select-String -Path $cfg -Pattern '<apikey>([^<]+)</apikey>' | ForEach-Object { $_.Matches[0].Groups[1].Value }
# UI port — may be 8385 if 8384 was occupied (e.g. by a VS Code SSH tunnel to the VPS)
Select-String -Path $cfg -Pattern 'address>127\.0\.0\.1:\d+' | Select-Object -First 1
```

Once you have `$api` and `$port`:

```powershell
$h = @{ "X-API-Key" = $api }
$base = "http://127.0.0.1:$port/rest"

# Get own Device ID
Invoke-RestMethod -Uri "$base/system/status" -Headers $h | Select-Object myID
```

## Onboarding flow

1. **Pre-flight:** confirm SyncthingWindowsSetup is installed and the service is running. If `netstat -ano | findstr :8384` shows an unexpected listener, it may be a VS Code Remote-SSH tunnel to the VPS (see PLAN-WINDOWS gotcha #4) — use the actual Windows Syncthing port (likely 8385) instead.
2. **Get your Device ID** (command above) and hand it to the VPS-side Claude.
3. **Add the VPS as a remote device** — the VPS-side Claude will also add you first; both sides need the entry. On your side, when adding the VPS:
   ```powershell
   $body = @{
     deviceID = "<VPS-DEVICE-ID>"
     name = "VPS"
     compression = "metadata"
     autoAcceptFolders = $false   # critical: prevents surprise-accept as sendreceive
     addresses = @("dynamic", "tcp://<VPS-IP>:22000")
   } | ConvertTo-Json
   Invoke-RestMethod -Method Put -Uri "$base/config/devices/<VPS-DEVICE-ID>" -Headers $h -ContentType "application/json" -Body $body
   ```
4. **Wait for the folder offer.** When the VPS shares the folder, Syncthing will *either* queue it as a pending offer (good) *or* auto-accept it as `sendreceive` (the installer-template quirk — see PLAN gotcha #7).
5. **Accept as Receive Only.** If the offer is pending, accept it via the UI with Advanced → Folder Type: `Receive Only`, path `C:\Users\<you>\ObsidianVault`. If it was auto-accepted, **immediately** patch:
   ```powershell
   $body = '{"type":"receiveonly"}'
   Invoke-RestMethod -Method Patch -Uri "$base/config/folders/obsidian-vault" -Headers $h -ContentType "application/json" -Body $body
   ```
6. **Verify clean state.** Both of these must hold:
   ```powershell
   Invoke-RestMethod -Uri "$base/config/folders/obsidian-vault" -Headers $h | Select-Object type
   # -> type : receiveonly
   Invoke-RestMethod -Uri "$base/db/status?folder=obsidian-vault" -Headers $h | Select-Object receiveOnlyTotalItems, receiveOnlyChangedBytes
   # -> both 0
   ```
   If `receiveOnlyTotalItems > 0`, the folder was briefly writable before the flag landed. Click **Revert Local Changes** in the UI (or `POST /db/revert?folder=obsidian-vault`) to clean up.
7. **Report back to the VPS-side Claude:** folder type, `receiveOnlyTotalItems`, connection status.
8. **Run the isolation test with them** (see next section).

## Isolation test protocol (run jointly with VPS-side Claude)

This verifies the read-only contract in actual behavior, not just config.

1. VPS-side Claude snapshots `/root/obsidian-vault/`.
2. Your turn: create `C:\Users\<you>\ObsidianVault\test-from-windows.md` with any content.
3. Wait ~30 seconds. Check your Syncthing UI: the folder must show **Local Additions** (non-zero); verify:
   ```powershell
   Invoke-RestMethod -Uri "$base/db/status?folder=obsidian-vault" -Headers $h | Select-Object receiveOnlyTotalItems
   # -> > 0
   ```
4. VPS-side Claude confirms the file did **not** appear on `/root/obsidian-vault/`. If it did, the test failed — folder type is wrong; stop and debug.
5. Click **Revert Local Changes** in your UI (or `POST /db/revert?folder=obsidian-vault`). The local file disappears.
6. Re-verify `receiveOnlyTotalItems: 0`.

**Pass criteria:** step 4 shows zero propagation from Windows to VPS. Any propagation = folder type drifted or was never applied.

## Day-to-day expectations for the user

- **Obsidian can open the vault and render it** (graph view included). All read-side features work.
- **Edits / new notes / deletes in Obsidian stay on Windows only** and are rolled back on the next Revert Local Changes pass. This is the intended contract — if the user wants to make a change, they coordinate with the VPS side.
- **Ignore patterns** from the VPS's `.stignore` apply here too — Obsidian UI state (`.obsidian/workspace*`, etc.) won't show up as Local Additions, so `receiveOnlyTotalItems` should stay at 0 during normal Obsidian use.

## What NOT to do

- Do not flip folder type back to `sendreceive` for convenience. Any such change is a silent two-writer bug waiting to happen.
- Do not `DELETE` the folder to "reset" it without coordinating with the VPS-side Claude — the initial resync is not free.
- Do not expose the Syncthing Web UI off-host. Default `127.0.0.1` binding is correct.
