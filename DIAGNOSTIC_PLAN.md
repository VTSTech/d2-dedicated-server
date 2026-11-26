# Long-Term Diagnostic Plan — D2 Dedicated Server + D2VP Client

## Purpose

Single, durable, multi-session diagnostic plan for the D2 private-realm project. Use it across
sessions so we stop re-deriving context and instead run the same checklist, capture evidence the
same way, and converge on the remaining blockers:

1. Recurring "server down" after char select / create (primary, server side).
2. Character list does not show after realm logon (client side, currently believed downstream of an
   MCP/BNet teardown).

## Evidence protocol (user-mandated — do not deviate)

- **Only read logs the user gives you.** Do not go hunting for logs (`/media/sf_!VBoxShared`,
  `192.168.0.14`, "random old logs"). There is weeks of dev history on the VM; assume anything found
  by us is stale.
- **Current boot only.** Anything not from the current VM boot is suspect.
- **Gateway:** current VM = `192.168.0.16`. `192.168.0.14` is old.
- **Version markers:** client `Rxx` values are a good indicator of how current the content is
  (`Vanilla+0.0.9-R243` seen). Record them.
- **Thread ids:** keep the tid map from the current log (`3324` BNet/main, `6216` BNCS connect,
  `7984` recv, `3564` close/download, `4380`/`6104`/`5180`/`7924`/`16388` MCP-socket ops).

## Long-term plan — phases

### Phase 0 — Server health baseline
- [x] `docker ps -a --format '{{.Names}} {{.State}}'` on all four services.
- [x] `docker logs deploy-realmd-1 | grep -E "getfiletime|realm logon|create|character|NO GS|SERVER_DOWN"`.
- [x] `curl -s localhost:8080/readyz`, `/healthz` (or equivalent) on realmd. → health port is **published as `18080:8080`** (container-internal 8080); both `/readyz` and `/healthz` now return **200 / "ready" / "ok"** on `127.0.0.1:18080`.
- [x] Redis: ping, `dbsize`, scan `realmd:*` → PONG, dbsize 14, keys under `realmd:*`.
- [x] GS logs: look for `cannot reach the realm store — retrying`, publish/fleet lines → **retry loop IS ACTIVE**.
- [x] e2e clientless run to prove server answers charlist + create when fleet is healthy → ran: stack not confirmed ready in 20s, login failed `LoginFailed`.

### Phase 1 — Server fleet / "server down"
- [ ] Fix GS redis retry loop (Priority #1 per handoff).
- [ ] Confirm fleet publishes; confirm realmd `NO GS available` → `CREATE_SERVER_DOWN` chain gone.

### Phase 2 — Client auth / realm logon
- [ ] Capture current `d2boot.log` (user-provided only).
- [ ] Verify SID sequence: `SID_AUTH_INFO(0x50)` → `AUTH_CHECK(0x51)` accepted → `SID_GETFILETIME(0x33)` →
       `SID_LOGONRESPONSE2(0x3A) result=0` → `SID_QUERYREALMS2(0x40)` → `SID_LOGONREALMEX(0x3E) realm=... OK`.
- [ ] Confirm no `45 s TIMEOUT`, no `aborted (protocol flagged dead)` during auth.

### Phase 3 — MCP charlist
- [ ] Confirm expected post-0x3E sequence (working /media run):
       `wsaref -> 2 (create)` → `MCP: 0x01 LOGINREQ sending len=70 account=...` → `MCP: recv 7 bytes` →
       `MCP: LOGINREPLY result=0 -> sending CHARLISTREQ (0x19)` → `MCP: pkt 0x19 ...` → char entries.
- [ ] Confirm CHARSEL state machine reaches state 0 (loaded) and `gnCharSelLoadedCharCount`.
- [ ] Confirm create-character availability flag.

### Phase 4 — Create / join
- [ ] `MCP: 0x03 CREATEGAME sent` → reply `result 0, game token` → `MCP: 0x04 JOINGAME sent` → GS IP.

### Phase 5 — Convergence
- [ ] Re-run e2e → client round trip.

## Per-session checklist

### Server
- [ ] Containers running (`deploy-realmd-1`, `deploy-gs-native-1`, `deploy-postgres-1`, `deploy-redis-1`).
- [ ] realmd `/readyz` + `/healthz` OK; bnftp serving `bnserver-D2DV.ini` + ToS.
- [ ] `REALMD_GAME_ADDR` = bare dotted quad (`127.0.0.1`).
- [ ] GS redis reachable; no `cannot reach the realm store — retrying`.
- [ ] e2e reaches create-game when fleet healthy.

### Client
- [ ] User pastes current `d2boot.log`.
- [ ] Annotated: gateway, Rxx, tid map, SID ops, MCP lines, wsaref, protocol-dead.
- [ ] CharSelect screen shown → SID 0x40 → 0x3E → MCP STARTUP/LOGINREQ → CHARLISTREQ → reply → display.
- [ ] Create-character button state.

### Synthesis
- [ ] Correlate server-side failure with client-side teardown.
- [ ] Record new "good" evidence for each stage.

## Failure-signature → suspect table (current knowledge)

| Signature in client log | Likely cause | Phase |
|---|---|---|
| `wait 0x3e -> reply arrived` then nothing MCP, then `SID PROTOCOL DEAD FLAG set` | MCP session never started/logged in; BNCS teardown | 2/3 |
| `wsaref -> 2 (create)` then `MCP: 0x01 LOGINREQ sending` then clean flow | Healthy path (known-good /media) | 3 |
| `45 s TIMEOUT (server never answered)` | Server/SID handler stall | 2 |
| `aborted (protocol flagged dead)` | Send refused / socket gone | 2/3 |
| `recv failed (...) -> protocol dead` | Socket/IO death | 2 |
| GS `cannot reach the realm store — retrying` | GS fleet-emptying retry loop | 1 |
| realmd `NO GS available` → `CREATE_SERVER_DOWN` | Fleet down | 1 |

## Open questions / next probes

1. Why did the current run produce no MCP lines between the 0x3E reply and the protocol death, while
   the /media run immediately logged `MCP: 0x01 LOGINREQ sending`?
2. Is the `gnMCPConnectThreadHandle` / pump short-circuit path leaving the CHARSEL charlist pump parked?
3. Is the protocol death itself a consequence (teardown) or the cause (i.e., who called `BNSEND_Disconnect`)?
4. Once server fleet is fixed, does the client char list populate and create become available?

## Status snapshot (2026-11-25)

- Server: bnftp ToS + `bnserver-D2DV.ini` serving fixed; `REALMD_GAME_ADDR` fix applied; GS redis retry
  loop still the prime server-side suspect.
- Client: auth + realm logon confirmed working (R243); char list/create still not reached in current run.
