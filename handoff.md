# Handoff for d2-dedicated-server Session

## Current State

**Server is running** via Docker Compose with the native GS profile:
```bash
cd deploy
docker compose --profile native up -d
```

**Container Status:**
- `deploy-realmd-1` - Running, listening on 6112 (bnet) and 4000 (game traffic)
- `deploy-gs-native-1` - Running, game server
- `deploy-postgres-1` - Running, persistence layer
- `deploy-redis-1` - Running, in-memory state

**What's Working:**
- ✅ Accounts can be created and logged in
- ✅ ToS file (tos-unicode_USA.txt) is being served via BNFTP
- ✅ Version check passes
- ✅ Realm logon succeeds
- ✅ Game server is running and registered
- ✅ `bnserver-D2DV.ini` IS found and served via BNFTP (getfiletime -> mtime, BNFTP serving 108 bytes)
- ✅ MCP charlist works (charlist requested=8 -> total=N returned=N)
- ✅ MCP char logon works (char logon '<char>' -> ok)
- ✅ MCP create game routes once the GS fleet is registered (create -> routed)
- ✅ MCP join game works (join -> took over the unconfirmed seat / gs route recorded)
- ✅ realmd starts cleanly (after REALMD_GAME_ADDR fix)
- ✅ e2e (stress-e2e) reaches create-game when the fleet is healthy
- ✅ Game-entry chain proven end to end with the current source: create -> join (seated/known) -> token
      resolve -> phase .waiting -> .played -> "ENTERED the world" -> in-world -> in-game (units > 0). Verified
      twice with a freshly rebuilt native GS after adding holdOne phase instrumentation.

**What's NOT Working:**
- ❌ Real D2VP client: character list still doesn't appear after realm logon (client-side display; the
      clientless e2e proves the server side answers charlist fine when the stack is healthy)

## Session Diagnosis Progress — 2026-11-25

**Closed Bnet status: DEFERRED by decision.** Focus is moving to OPEN Bnet and the Pi arm64 D2GS.

**This session's findings (closed-Bnet path):**
- BNFTP serving fixed (`getfiletime -> mtime`, ToS 1315 B, `bnserver-D2DV.ini` 108 B).
- Version fix applied: `REALMD_CLIENT_VERSIONS: "vb:14=1.14d"` in `deploy/compose.yaml` -> realmd now logs
  `client engine 1.14d`, `startup session=... -> account=... engine=1.14d`, and char create comes back
  `engine=1.14d`.
- GS redis retry loop: root-caused to compose bring-up/DNS churn; clean bring-up fixed it. Fleet publishes
  again, no retry loop, no `NO GS` -> `CREATE_SERVER_DOWN`.
- e2e: charlist answered (`requested=8 -> total=N returned=N`), char logon ok, create-game routed,
  join seated, game edge spliced.
- **`NEVER ENTERED: NeverReady` world-entry failure — RESOLVED this session.** Added phase instrumentation
  to `holdOne()` in `apps/d2gs-native/realm.zig`:
  - `if (s.phase == .waiting and s.join_resolved) { s.phase = .played; note("... ENTERED the world (join_resolved was set) clients={d}\n", ...); }`
  - else-branch logs `... still waiting clients={d} join_resolved={} ...` while `phase == .waiting`.
  - Rebuilt native GS image (`docker compose -f deploy/compose.yaml -f deploy/compose.e2e.yaml --profile
    native build gs-native`; two transient `NameServerFailure` DNS failures before success), force-recreated
    `deploy-gs-native-1`, re-ran e2e twice -> both clean (`round 1 1/1 in-game ok`, world `CreatePlayer` +
    `LoadAct` + `units>0`).
- Live D2VP client: opened Bnet, created a game, **shared stash even works** -> the closed-bnet / create /
  join / stash path is functionally working on the server side.

## The Core Issue

**Problem:** The volume mount on `/data` is replacing the entire directory including `/data/bnftp`, which contains the `bnserver-D2DV.ini` file that needs to be served.

**Current Volume Setup:**
```yaml
volumes:
  - realmd-data:/data          # This replaces everything including /data/bnftp
  - ./apps/realmd/assets/bnftp:/data/bnftp  # This bind mount should work but doesn't
```

**Dockerfile Strategy:**
- Files are copied to `/data/bnftp/bnserver-D2DV.ini` in the `build-realmd` stage
- The `realmd` stage (FROM scratch) copies this to `/data/bnftp/`
- But the volume mount on `/data` replaces the entire directory

## What We Tried

1. **Copied file in build stage** - Added COPY in `build-realmd` stage to copy `bnserver-D2DV.ini` to `/data/bnftp/`
2. **Added entrypoint script** - Tried to copy files from image to volume at runtime (didn't work due to FROM scratch)
3. **Changed COPY location** - Tried different COPY paths in Dockerfile

## Next Steps

### 1. Fix the GS redis retry loop (Priority #1 — the recurring "server down" root cause)

**Option A:** Mount only `/data/bnftp` as a volume, keep `/data` as a bind mount
```yaml
volumes:
  - ./apps/realmd/assets/bnftp:/data/bnftp:ro
```

**Option B:** Don't use a volume for `/data/bnftp` at all, just bind mount it
```yaml
volumes:
  - realmd-data:/data
  - ./apps/realmd/assets/bnftp:/data/bnftp
```

**Option C:** Remove the bnftp files from the Docker image entirely and rely on the bind mount

### 2. Re-run the e2e until it passes

Once the GS fleet is healthy again, run the clientless e2e against the live stack:
`docker run --rm --network deploy_default deploy-stress-e2e:latest --host realmd --rounds 1 --clients 1
--runs 1 --account <acct> --chars <char>:1 --wait-ms 20000`.

Known state-pollution gotchas: game-name registry (`realmd:game:*`) and char locks persist in redis/PG
across `restart`; use `docker compose -f deploy/compose.yaml down -v` for a real wipe, and use a fresh
account + fresh char per run (the probe holds the char via its game).

### 3. Test with D2VP Client — DEFERRED (closed Bnet)

Closed Bnet work is **deferred** (user decision). The stack-side closed-bnet path is now verified working:
bnftp serving ✅, version check ✅, realm logon ✅, charlist ✅, create/join ✅, world-entry ✅ (fresh native-GS
build + holdOne phase instrumentation, e2e green twice), live D2VP created a game and the shared stash works.
The only remaining closed-bnet-adjacent item is the real-D2VP-client character-list display.

### 4. Open Bnet

1. Repeat the same client smoke (connect, logon, charlist, create, join, stash) against the **open** bnet
   endpoint / realm configuration rather than the closed one.
2. Compare logs (bncs/d2cs/bnetd + D2VP `d2boot.log`) against the closed run to isolate what differs.

### 5. Pi arm64 D2GS (future goal)

1. Cross-build the native GS (`-Dtarget=aarch64-linux-...`) and bring up under the Pi.
2. Reuse the verified native-GS logic (redis store client, QSERVER patches) on arm64.

### 6. Re-validate e2e on the way back

`docker compose -f deploy/compose.yaml -f deploy/compose.e2e.yaml --profile native run --rm stress-e2e
--host realmd --rounds 1 --clients 1 --runs 1` (build took ~7 min; transient `NameServerFailure` DNS flakes
seen twice before success).

## Important Files

**Dockerfile:** `deploy/Dockerfile`
- Line ~90: COPY command for bnftp files in build stage
- Line ~275: realmd stage COPY from build-realmd

**Compose File:** `deploy/compose.yaml`
- Line ~35: Volume mounts for realmd service

**Assets:** `apps/realmd/assets/bnftp/tos-unicode_USA.txt`
**Assets:** `apps/realmd/assets/bnftp/bnserver-D2DV.ini`

## Logs to Check

```bash
# Check realmd logs
docker logs deploy-realmd-1 2>&1 | tail -100

# Look for these messages:
# - "getfiletime 'bnserver-D2DV.ini' -> not held" (BAD - file not found)
# - "getfiletime 'bnserver-D2DV.ini' -> mtime X" (GOOD - file found)
# - "realm logon account=X realm=TypeGuru" (should appear)
# - "create account" (should work)
# - "character list" (currently missing)
```

## Debug Commands

```bash
# Check if file is accessible
docker run --rm -v deploy_realmd-data:/data alpine ls -la /data/bnftp/

# Check container mounts
docker inspect deploy-realmd-1 --format='{{json .Mounts}}'

# View realmd logs
docker logs -f deploy-realmd-1

# Restart services
cd deploy && docker compose restart realmd

# Rebuild (if needed)
cd deploy && docker compose build --no-cache realmd
```

## Environment

**Working Directory:** `/home/vtstech/workspace/d2-dedicated-server`
**Zig Version:** 0.14.1
**Docker Version:** 29.1.3
**Docker Compose Version:** 2.40.3+ds1-0ubuntu1
**Branch:** main (VTSTech fork)

## Notes

- The `bnserver-D2DV.ini` contains gateway configuration for the realm
- The file path in realmd is `<data_dir>/bnftp/<filename>` (not just `<data_dir>/`)
- Volume mounts in Docker Compose replace directories, they don't merge them
- The bind mount `./apps/realmd/assets/bnftp:/data/bnftp` should work, but we need to verify the files are actually there

## User Context

**VTSTech** is working on:
- Linux x64 native D2GS
- Pi arm64 D2GS (future goal)
- Custom D2VP client (LoD 1.14d with QoL features)
- Testing both Retail Game.exe and D2VP.exe

**Current Issue:** Closed Bnet shows "server down" after character select when trying to create a game.

**Update (this session):** the bnftp "not held" problem was fixed (bind mount + Dockerfile COPY verified:
`getfiletime -> mtime`, `BNFTP serving 'bnserver-D2DV.ini' (108 bytes)`, ToS 1315 bytes). The realmd startup
crash (`FATAL REALMD_GAME_ADDR 'gs-native:4000' did not resolve to an IPv4` -> exit) was fixed by setting
`REALMD_GAME_ADDR: "127.0.0.1"` in `deploy/compose.yaml` (config requires a bare dotted-quad; `net.resolve4`
rejects hostnames-with-port). With that, realmd boots, bnftp serves, logon + charlist + create-route all
verified live.

**Update (this session, continued):** version fix (`REALMD_CLIENT_VERSIONS: "vb:14=1.14d"`) -> client engine
now resolves to `1.14d` (was unmapped -> empty engine -> char create/join/world-entry stalling). GS redis
retry loop root-caused to compose bring-up/DNS churn; clean bring-up fixed it. Added holdOne phase
instrumentation in `apps/d2gs-native/realm.zig` (`.waiting` -> `.played` on `join_resolved`) + rebuilt native
GS image; e2e green twice in a row, world-entry now confirmed (`ENTERED the world ... in-game ok`).
**`NEVER ENTERED: NeverReady` RESOLVED.** Live D2VP client: opened Bnet, created a game, shared stash works.

**Direction (user decision):** closed Bnet investigation is **DEFERRED** — focus is now **OPEN Bnet** and the
**Pi arm64 D2GS** (future goal). See "Session Diagnosis Progress" and Next Steps §4-§5.
