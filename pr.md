# Code review fixes — EPICS Base

A full code review of EPICS Base (`libcom`, `database`, `ca`) surfaced 10
genuine, high-confidence bugs. Each fix is on its own branch off `7.0` so it
can be opened as a separate PR against the main project. All 10 together
compile cleanly (`make`, darwin-aarch64, no new warnings).

| # | Branch | Severity | Area |
|---|--------|----------|------|
| 1 | `histogram-add-count-oob-bin-index` | High | std/rec |
| 2 | `printf-ls-nul-terminator-off-by-one` | Medium | std/rec |
| 3 | `fix-ringbytes-reset-highwatermark-sign` | Low | libcom/ring |
| 4 | `fix-epicstimeaddseconds-negative-normalize` | Low | libcom/osi |
| 5 | `fix-errlog-resize-oob-printbuf-copy` | Low–Med | libcom/error |
| 6 | `fix-iocsh-on-error-wait-inverted-parse` | Medium | libcom/iocsh |
| 7 | `fix-putuint64string-truncation` | High | database/db |
| 8 | `fix-dbgetstringnum-uint64-cast` | High | database/dbStatic |
| 9 | `fix-dbreportdeviceconfig-overflow` | Medium | database/dbStatic |
| 10 | `fix-dbjlinkmapall-assignment` | Medium | database/db |

Each branch contains exactly one commit with the fix described below.

---

## 1. histogramRecord: heap OOB write when SGNL approaches ULIM — High
`modules/database/src/std/rec/histogramRecord.c`

`add_count()` finds the bin with a loop that assumes `nelm*wdth ==
ulim-llim`, but `wdth` is a rounded quotient, so `nelm*wdth` can be strictly
less than `ulim-llim`. A `SGNL` value just below `ULIM` leaves the loop with
`i == nelm+1`, and `pdest = bptr + nelm` writes one `epicsUInt32` past the
`nelm`-element buffer. Reachable over Channel Access via `caput HIST.SGNL`
(the field has `special(SPC_MOD)`). Fix: clamp `i` to `nelm`.

## 2. printfRecord: one-byte VAL overflow in `%ls` — Medium
`modules/database/src/std/rec/printfRecord.c`

In the long-string branch `n = vspace + 1` and `dbGetLink(DBR_CHAR)` fills
up to `n` chars without reserving a terminator, so `pval[n] = 0` writes at
`pval[vspace+1]`, one byte past the `sizv`-byte `VAL`
(`pval + vspace + 1 == prec->val + sizv`). Triggered by `FMT="%ls"` with a
source ≥ `vspace` chars. Fix: read at most `vspace` (and `precision`, not
`precision+1`) characters.

## 3. epicsRingBytes: reversed operands in ResetHighWaterMark — Low
`modules/libcom/src/ring/epicsRingBytes.c`

`epicsRingBytesResetHighWaterMark` computed used bytes as `nextGet -
nextPut`, the inverse of every other function in the file. For a ring
holding `u` bytes it stored `size-u`; since `Put` only ever raises the mark,
the high-water mark stayed permanently inflated. Diagnostic-only. Fix: use
`nextPut - nextGet`.

## 4. epicsTime: bad normalization of pre-epoch results — Low
`modules/libcom/src/osi/epicsTime.cpp`

When the accumulated nanoseconds went negative, `epicsTimeAddSeconds`
truncated the seconds toward zero while taking `|nsec| % nSecPerSec` for the
fraction, yielding an inconsistent `(seconds, nsec)` pair off by up to ~1 s.
Only reachable for pre-1990 results. Fix: floored division with a borrow so
the fraction stays in `[0, nSecPerSec)`.

## 5. errlog: OOB read when resizing the print buffer — Low–Medium
`modules/libcom/src/error/errlog.c`

`errlogBufResize` copied the print buffer from `&pvt.print->base` (the
address of the `char*` struct field) instead of `pvt.print->base` (the
buffer), reading `bufSize` bytes from an ~8-byte field — UB, and crashes
under sanitizers. Fix: drop the stray `&`.

## 6. iocsh: inverted parse test in `on error wait <delay>` — Medium
`modules/libcom/src/iocsh/iocsh.cpp`

`epicsParseDouble` returns 0 on success, but the success/failure branches
were swapped: a valid delay printed `Invalid 'on error wait' delay` and the
usage text, while an invalid delay was silently accepted with `timeout`
forced to 5.0. Fix: report the error on parse failure, accept on success.

## 7. dbConvert: UInt64→String truncated to 32 bits — High
`modules/database/src/ioc/db/dbConvert.c`

`putUInt64String` used `cvtUlongToString` (= `cvtUInt32ToString`), narrowing
the 64-bit source to its low 32 bits, so any value ≥ 2³² was corrupted (e.g.
`0x1_0000_0007` → `"7"`) on a `dbPut` of DBR_UINT64 into a string field.
Fix: use `cvtUInt64ToString`.

## 8. dbStaticLib: dbGetStringNum reads UINT64 through a 32-bit pointer — High
`modules/database/src/ioc/dbStatic/dbStaticLib.c`

The `DBF_UINT64` case dereferenced the 8-byte field as `*(epicsUInt32 *)`,
so large UINT64 values printed wrong and save/restore round-tripping broke
(hit by `dbWriteRecord`, `dbpr`, CA DBR_STRING reads). Fix: cast to
`epicsUInt64*`, matching the `DBF_INT64` case.

## 9. dbStaticLib: stack overflows in dbReportDeviceConfig — Medium
`modules/database/src/ioc/dbStatic/dbStaticLib.c`

`dbGetString()` returns up to 275 chars, but DTYP was copied into
`dtypValue[50]` and `"cvt(EGUL,EGUF)"` was built in `cvtValue[40]` with
`strcpy`/`strcat`. A long DTYP choice or high-precision EGUL/EGUF overflows
these stack buffers. Fix: size the buffers from `messagesize`.

## 10. dbJLink: assignment instead of comparison in dbJLinkMapAll — Medium
`modules/database/src/ioc/db/dbJLink.c`

The guard used `recname[0] = '\0'` (assignment) instead of `==`: it wrote a
NUL into the caller's string (truncating it), evaluated to 0, and left
`recname` non-NULL pointing at `""`, so every record failed the name match
and nothing was mapped. Fix: use `==`, matching the sibling `dbjlr`.
