# VPS Claude — Linux Client Onboarding

> **Paired instruction file:** [`local-claude.md`](./local-claude.md) — runbook for the Claude instance on the Linux client. Read it first so you know what the other side will be doing and what signals to expect.
> **Architecture context:** [`../PLAN-LINUX.md`](../PLAN-LINUX.md) — full setup plan, including the role model.

## Your role

You are running on the **VPS**. The VPS is the **sole writer** to `/root/obsidian-vault/`. Any Linux client is a **read-only mirror**. Your job is to keep the VPS side correct and help the Linux-side Claude get onto the share as receive-only.

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
CONFIG=/root/.local/state/syncthing/config.xml
API=$(grep -oP '<apikey>\K[^<]+' "$CONFIG")
BASE=http://127.0.0.1:8384/rest

# Verify folder role
curl -sH "X-API-Key: $API" "$BASE/config/folders/obsidian-vault" | jq '{type, devices: [.devices[].deviceID[:7]]}'

# Patch to sendonly (idempotent)
curl -sX PATCH -H "X-API-Key: $API" -H "Content-Type: application/json" \
  -d '{"type":"sendonly"}' "$BASE/config/folders/obsidian-vault"

# Connection status for a specific device
curl -sH "X-API-Key: $API" "$BASE/system/connections" | jq '.connections'

# Folder status (globalFiles, localFiles, needBytes, etc.)
curl -sH "X-API-Key: $API" "$BASE/db/status?folder=obsidian-vault"
```

## Onboarding flow (when a new Linux client joins)

1. **Pre-flight:** confirm the VPS invariants above. Patch any drift. Do not proceed otherwise.
2. **Wait for Device ID** from the Linux-side Claude (see step 2 of `local-claude.md`).
3. **Register the Linux device** with `autoAcceptFolders: false` explicitly set:
   ```bash
   curl -sX PUT -H "X-API-Key: $API" -H "Content-Type: application/json" \
     -d '{"deviceID":"<LINUX-ID>","name":"Linux","compression":"metadata","autoAcceptFolders":false,"addresses":["dynamic"]}' \
     "$BASE/config/devices/<LINUX-ID>"
   ```
4. **Add the device to the folder's share list:**
   ```bash
   # Fetch current device list first, then PATCH with the appended entry; do not replace wholesale.
   curl -sH "X-API-Key: $API" "$BASE/config/folders/obsidian-vault" \
     | jq '.devices += [{"deviceID":"<LINUX-ID>"}]' \
     | curl -sX PUT -H "X-API-Key: $API" -H "Content-Type: application/json" -d @- \
       "$BASE/config/folders/obsidian-vault"
   ```
5. **Confirm connection established** (`"connected": true` via `/system/connections`). If not, check that TCP 22000 is reachable from the client side and that global discovery is working.
6. **Wait for the Linux-side Claude to confirm** folder type is `receiveonly` and `receiveOnlyTotalItems: 0`. If nonzero, coordinate a Revert Local Changes on their side before running the isolation test.
7. **Run the isolation test with them** (see next section).

## Isolation test protocol (run jointly with Linux-side Claude)

This verifies the read-only contract in actual behavior, not just config.

1. Your turn: `ls /root/obsidian-vault/` — snapshot the current file list. Report it.
2. Linux side creates `test-from-linux.md` with some content.
3. Wait ~30 seconds.
4. Your side: re-list `/root/obsidian-vault/` — the new file must **not** appear. Also:
   ```bash
   curl -sH "X-API-Key: $API" "$BASE/db/status?folder=obsidian-vault" | jq '.globalFiles, .localFiles'
   ```
   Counts should be unchanged from pre-test.
5. Linux side reverts local changes. Confirm with them the file is gone from their machine.
6. Final check on VPS: nothing changed, nothing in `.stversions/` from the test.

**Pass criteria:** step 4 shows zero propagation from client to VPS. Any propagation = folder type drifted or was never applied. Fix before marking the setup done.

## When to push a change to clients

Writes to `/root/obsidian-vault/` (from you, the user, or Claude memory) propagate automatically. No extra action needed.

`.stignore` edits propagate on the next folder scan.

## What NOT to do

- Do not change folder type back to `sendreceive`. If a Linux client needs write access, that's a separate repo/vault, not this one.
- Do not `DELETE` a client device to "reset" it without coordinating with the Linux-side Claude.
- Do not expose `127.0.0.1:8384` to the public internet. SSH port-forward for remote UI access.
