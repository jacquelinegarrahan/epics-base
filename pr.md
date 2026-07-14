# Security scan fixes — EPICS Base

Branch: `security-scan-base` (off `7.0`)

A comprehensive security scan of EPICS Base focused on the network-facing
attack surface (the Channel Access client and server, which speak an
unauthenticated protocol over TCP/UDP) and the local file/config parsers.
This PR fixes every confirmed memory-safety and fail-state issue found. All
changes compile cleanly (`make`, darwin-aarch64, no new warnings).

Threat model for the "remote" items: an unauthenticated CA peer. A default
IOC (no access-security config loaded) grants write access to everyone, so
the server-side write path is reachable pre-auth; the client-side paths are
reachable from any server the client connects to (or an on-path attacker on
the circuit / a spoofed UDP responder).

## Critical

### 1. rsrv: unvalidated `m_count` → remote heap OOB read+write
`modules/database/src/ioc/rsrv/camessage.c`

A CA data message is framed on `m_postsize`, but the in-place byte-order
conversion (`caNetConvert`) and the database put are driven by the
independent wire field `m_count`. `write_action`/`write_notify_action`
never checked that `m_count` elements actually fit in `m_postsize` bytes, so
a single small `CA_PROTO_WRITE` with a huge `m_count` made `caNetConvert`
byte-swap far past the fixed receive buffer (≈500 KB+ OOB, unbounded with
the extended header). Added `payloadTooSmall()`, computed in 64-bit
(`dbr_size_n` truncates to 32-bit while the converter loops the full count),
and reject with `ECA_BADCOUNT` before conversion in both handlers.

### 2. CA client: unvalidated response `m_count` → heap OOB write
`modules/ca/src/client/cac.cpp`, `getCopy.cpp`, `syncGroupReadNotify.cpp`,
`syncGroup.h`

Mirror of #1 on the client. `readNotifyRespAction`/`eventRespAction`/
`readRespAction` fed the server's `m_count` straight into the in-place
`caNetConvert`, overflowing `pCurData` on *ordinary monitor updates*. Added
`responsePayloadOK()` (64-bit) and reject with `ECA_BADCOUNT` before
conversion. Also clamped the copy-out in `getCopy::completion` and
`syncGroupReadNotify::completion` to the caller's requested count (the
latter now stores that count).

### 3. CA client: OOB function-pointer call in `exceptionRespAction`
`modules/ca/src/client/cac.cpp`

The exception dispatcher bounds-checked `hdr.m_cmmd` (always
`CA_PROTO_ERROR`, in range) but indexed `tcpExcepJumpTableCAC` with the
attacker-controlled embedded `req.m_cmmd` (0–65535, table has 28 entries),
then called through the resulting garbage pointer — a control-flow-hijack
primitive. Now bounds-checks `req.m_cmmd`, the value actually used.

## High

### 4. calc: `postfix()` translation-stack overflow
`modules/libcom/src/calc/postfix.c`

The infix→postfix converter pushes operators onto a fixed `stack[80]` with
no bound check; an open paren consumes a stack slot without advancing the
runtime-depth counter that is checked, so ≥80 nested parens smash the stack.
Reachable by writing a `CALC`/`OCAL` field (network-writable). Added a
capacity guard (`CALC_ERR_OVERFLOW`) at each push site. ASan-confirmed
before/after.

### 5. CA client: enum `no_str` not clamped
`modules/ca/src/client/convert.cpp`, `tool_lib.c`

`no_str` from a `dbr_gr_enum`/`dbr_ctrl_enum` was byte-swapped but never
clamped; downstream consumers iterate `strs[0..no_str-1]` over a 16-entry
array. Clamped to `MAX_ENUM_STATES` centrally in the converters, plus a
defense-in-depth clamp at the `caget`/`caput` print site.

## Medium

### 6. exception context strings reach `%s` unterminated
`modules/ca/src/client/cac.cpp`, `udpiiu.cpp` — bound the wire context
string to the bytes actually present (`%.*s` / bounded copy) so a
non-NUL-terminated payload can't over-read.

### 7. dbStaticLib: undersized CALC validation buffer
`modules/database/src/ioc/dbStatic/dbStaticLib.c` — `RPCL_LEN` was sized for
an 80-char infix but `CALC`-type fields hold 160, so `postfix()` output
could overflow. Now sized from `pflddes->size` (heap, freed). ASan-confirmed.

### 8. dbLoadTemplate: `sub_collect` unbounded appends
`modules/database/src/ioc/dbtemplate/dbLoadTemplate.y` — the substitution
accumulator used unbounded `strcat` of lexer tokens / the command-line macro
string into a fixed buffer. Added a bounds-checked `sub_append()` helper and
a length check on the macro string. (Local `.substitutions`/macro input.)

### 9. asLib lexer: unbounded `sprintf`
`modules/libcom/src/as/asLib_lex.l` — an over-range numeric literal in a
`.acf` file formatted an unbounded `yytext` into `message[40]`. Switched to
`epicsSnprintf`. (Local access-security config.)

### 10. iocLogServer: `getDirectory()` stack overflow
`modules/libcom/src/log/iocLogServer.c` — `strcpy` of a filename (up to 511
chars from `$EPICS_IOC_LOG_FILE_NAME`) into `temp[256]`. Switched to bounded
`epicsSnprintf`. (Local environment.)

## Low

### 11. rsrv: `event_add_action` short-payload OOB read
`modules/database/src/ioc/rsrv/camessage.c` — read `mon_info.m_mask` without
checking `m_postsize >= sizeof(struct mon_info)`. Now checked.

### 12. Access security: `asShutdown` leaves `asActive` TRUE
`modules/database/src/ioc/as/asDbLib.c` — after freeing/NULLing `pasbase`,
`asActive` stayed TRUE, breaking the `asActive ⇒ pasbase != NULL` invariant
(latent NULL-deref / stale access on in-process IOC restart). Now clears
`asActive`.

## Verified and NOT changed (checked, no bug)

The remotely-reachable JSON channel-filter path (`rsrv` → `dbChannelCreate`
→ bundled YAJL) was audited and is well-hardened: the parser is iterative
(no stack-exhaustion), channel names are NUL-terminated by rsrv, and the
`arr` filter index math is bounds-checked. The access-security enforcement
points (every CA put/put-notify/read gated by `asCheckPut`/`asCheckGet`)
were traced and are correct. The `read_action` send path is safe because the
send buffer is sized to `m_count` and `dbGet` clamps to `no_elements`.
