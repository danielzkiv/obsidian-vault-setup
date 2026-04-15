# VPS Claude — Windows Client Onboarding

> **Paired instruction file:** [`local-claude.md`](./local-claude.md) — runbook for the Claude instance on the Windows machine. Read it first so you know what the other side will be doing and what signals to expect.
> **Architecture context:** [`../PLAN-WINDOWS.md`](../PLAN-WINDOWS.md) — full setup plan, including the role model and prior execution notes.

## Your role

You are running on the **VPS**. The VPS is the **sole writer** to `/root/obsidian-vault/`. The Windows client is a **read-only mirror**. Your job is to keep the VPS side correct and help the Windows-side Claude get onto the share as receive-only.

## VPS-side invariants to maintain

| Setting | Required value | Why |
|---|---|---|
| Folder `obsidian-vault` type | `sendonly` | VPS-only writes propagate out; nothing from clients is accepted |
| `.stignore` content | `.obsidian/workspace*`, `.obsidian/cache*`, `.obsidian/graph.json`, `.obsidian/app.json`, `.trash/` | Ignore patterns propagate to clients; keeps Obsidian per-device UI state from flagging as "Local Additions" on mirrors |
| Web UI bind | `127.0.0.1:8384` only | Never exposed to the internet; reconfigure via SSH tunnel |
| Versioning | `simple`, keep 5 | Safety net for accidental VPS-side deletes |
| New devices | `autoAcceptFolders: false` | Prevents surprise-accept on client-side quirks |

## API cheatsheet

```bash
# Config + API key live here on this VPS
CONFIG=/root/.local/state/syncthing/config.xml
API=$(grep -oP '<apikey>\K[^<]+' "$CONFIG")
BASE=http://127.0.0.1:8384/rest

# Verify folder role
curl -sH "X-API-Key: $API" "$BASE/config/folders/obsidian-vault" | jq '{type, devices: [.devices[].deviceID[:7]]}'

# Patch to sendonly (idempotent)
curl -sX PATCH -H "X-API-Key: $API" -H "Content-Type: application/json" \
  -d '{"type":"sendonly"}' "$BASE/config/folders/obsidian-vault"

# Connection status for a device
curl -sH "X-API-Key: $API" "$BASE/system/connections" | jq '.connections | to_entries[] | select(.key|startswith("SC4OMBT")) | .value | {connected, address}'

# Folder status (globalFiles, localFiles, needBytes, etc.)
curl -sH "X-API-Key: $API" "$BASE/db/status?folder=obsidian-vault"
```

## Onboarding flow (when a new Windows client joins)

1. **Pre-flight:** confirm the VPS invariants above. Patch any drift. Do not proceed otherwise.
2. **Wait for Device ID** from the Windows-side Claude (they will hand it to you — see step 3 of `local-claude.md`).
3. **Register the Windows device** with `autoAcceptFolders: false` explicitly set:
   ```bash
   curl -sX PUT -H "X-API-Key: $API" -H "Content-Type: application/json" \
     -d '{"deviceID":"<WINDOWS-ID>","name":"Windows","compression":"metadata","autoAcceptFolders":false,"addresses":["dynamic"]}' \
     "$BASE/config/devices/<WINDOWS-ID>"
   ```
4. **Add the device to the folder's share list:**
   ```bash
   curl -sX PATCH -H "X-API-Key: $API" -H "Content-Type: application/json" \
     -d '{"devices":[{"deviceID":"BVVAFG3-..."},{"deviceID":"<WINDOWS-ID>"}]}' \
     "$BASE/config/folders/obsidian-vault"
   ```
   (Keep the VPS's own device ID in the list; replacing wipes it.)
5. **Confirm connection** is established (`"connected": true` via `/system/connections`). If not, check that TCP 22000 is reachable from the client side.
6. **Wait for the Windows-side Claude to confirm** they set folder type to `receiveonly` and report `receiveOnlyTotalItems: 0`. If they report a nonzero value, the folder was briefly writable before the flag landed — coordinate a Revert Local Changes on their side.
7. **Run the isolation test with them** (see the test section below).

## Isolation test protocol (run jointly with Windows-side Claude)

This verifies the read-only contract in actual behavior, not just config.

1. Your turn: `ls /root/obsidian-vault/` — snapshot the current file list. Report it.
2. Windows side creates `test-from-windows.md` with some content.
3. Wait ~30 seconds.
4. Your side: re-list `/root/obsidian-vault/` — the new file must **not** appear. Also:
   ```bash
   curl -sH "X-API-Key: $API" "$BASE/db/status?folder=obsidian-vault" | jq '.globalFiles, .localFiles'
   ```
   Counts should be unchanged from pre-test.
5. Windows side clicks **Revert Local Changes**. Confirm with them the file is gone from their machine.
6. Final check on VPS: nothing changed, nothing in `.stversions/` from the test.

**Pass criteria:** step 4 shows zero propagation from client to VPS. Any propagation = folder type drifted or was never applied. Fix before marking the setup done.

## When to push a change to clients

Because the VPS is authoritative, routine notes + Claude memory writes are already covered — just write to `/root/obsidian-vault/` and Syncthing handles the rest. No extra action needed.

If you need to push a `.stignore` change, edit the file in place; it propagates to clients automatically with the next scan.

## What NOT to do

- Do not change folder type back to `sendreceive` for convenience. If a client needs to edit, they work in a separate branch/repo — not this vault.
- Do not `DELETE` the device to "reset" a stuck client without coordinating with the Windows-side Claude first. The client can be re-added cheaply but the initial sync is not free.
- Do not expose `127.0.0.1:8384` to the public internet. Use SSH port-forward (`ssh -L 8384:localhost:8384 root@vps`) for remote UI access.
