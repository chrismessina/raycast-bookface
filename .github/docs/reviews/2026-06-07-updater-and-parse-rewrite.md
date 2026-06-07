# Review: yc-update action, parseYcJson rewrite, account/Me changes

**Date:** 2026-06-07
**Branch / PR:** main (pre-initial-commit; all files untracked)
**Reviewer:** /code-review (xhigh, 9 angles + verify + sweep)
**Status:** partially-addressed

Scope: the changes made in the 2026-06-07 session — `src/views/updater.tsx` (new),
`src/lib/yc.ts` (`parseYcJson` rewrite + JSON-recovery helpers), `src/lib/types.ts`
(`Me` extended), `src/account.tsx` (email/companies + Update action), `src/lib/items.tsx`
(Update action wiring), `src/lib/empty-states.tsx` (CopyLogin title fix).

## Findings

### [F1] runYc catch-path sentinel-matched auth phrases against data — non-blocking
**Where:** src/lib/yc.ts:174 (catch block, non-zero-exit path)
**Issue:** `isUnauthedMessage(stderr + " " + stdout)` re-applied the same pattern-on-data
bug that `parseYcJson` was rewritten to fix — a non-auth failure whose output contains
"401"/"unauthorized"/"yc login" would be misclassified as not-authed for `ask`/`account`.
**Evidence:** Verifier CONFIRMED mechanism; test `classify("", JSON.stringify({response:"HTTP 401 unauthorized"}))` returned "not-authed" under old logic.
**Resolution:** FIXED this session. Added `looksLikeJson()` guard; the catch now only
sentinel-matches output that is not a recoverable JSON payload. Re-test: 4/4 pass
(genuine auth → not-authed; JSON-data-mentioning-401 → error).

### [F2] extractJsonPayload anchored on the first bracket anywhere — non-blocking
**Where:** src/lib/yc.ts:83 (old `extractJsonPayload`)
**Issue:** A bracketed chatter prefix (e.g. `[notice] …\n[{realdata}]`) made the scanner
return the wrong slice ("[notice]"), failing recovery (or returning wrong data).
**Evidence:** Reproduced — `extract("[notice] …\n[{\"id\":1}]")` returned `"[notice]"`,
`JSON.parse` threw, recovery defeated. Documented case ("Token expired, refreshing…", no
brackets) was unaffected. PLAUSIBLE (no evidence yc emits bracketed chatter).
**Resolution:** FIXED this session. Replaced with `tryParseEmbeddedJson()` which scans every
`{`/`[` position and returns the first slice that actually `JSON.parse`s. Re-test passes
the bracketed-chatter case.

### [F3] summarizeUpdate brittle string-match; empty output → false "updated" — non-blocking
**Where:** src/views/updater.tsx:23-30
**Issue:** `!/already up to date/i.test(output)` treats empty/blank output as an upgrade, and
depends on the CLI's exact human phrasing. If yc changes wording or prints nothing, a no-op
run is labeled "YC CLI updated".
**Evidence:** Confirmed by read; `!/.../.test("")` === true. Low impact — real `yc update`
always prints "Current version: X\nAlready up to date." (verified live, exit 0, on stdout).
**Resolution:** FIXED this session. Replaced string-sniffing with a version diff: read `yc -v`
before, run `yc update`, read `yc -v` after; report "Updated to X" only when before !== after,
else "Already up to date". Removed `summarizeUpdate` and the stdout+stderr concatenation.

### [F4] runUpdate had no re-entrancy guard — non-blocking
**Where:** src/views/updater.tsx:51
**Issue:** The Update action stayed enabled during an update; a fast double-Enter could launch
two concurrent `yc update` processes both rewriting the binary in place.
**Evidence:** Verifier PLAUSIBLE — `onAction={runUpdate}` never disabled, no guard; narrow race
window; low harm (self-update of a symlinked binary).
**Resolution:** FIXED this session. Added a synchronous `useRef` (`inFlight`) guard that blocks
re-entry before the async state commits; cleared in `finally`.

### [F5] parseVersion unanchored regex — non-blocking
**Where:** src/views/updater.tsx:16
**Issue:** `/\d+\.\d+\.\d+/` matches the first dotted-triple anywhere; if `yc -v` ever prints a
banner with another version/date, it returns the wrong token.
**Evidence:** Confirmed by read. Safe today — `yc -v` prints bare "0.0.8" on stdout (verified
live). Fragile if the banner format changes.
**Resolution:** OPEN (deferred — no live trigger; revisit if yc -v output changes).

## Dismissed after verification (recorded so they aren't re-raised)

- **stderr drop in useExec parseOutput** (updater.tsx:43) — REFUTED: `yc -v` and `yc update`
  both write to **stdout**, stderr empty (verified: `yc -v 1>/dev/null` prints nothing). Version
  display works.
- **parse-first scalar bypass** (`401`/`null` as whole stdout) — REFUTED: yc `--json` emits
  objects/arrays; auth errors are non-JSON plain text.
- **company_id === 0 falsy check** (items.tsx:226) — real but **pre-existing unchanged code**,
  not introduced this session. Track separately if ever touched.
- **setState-after-unmount across the 120s await** — benign under React 18 (no warning; useExec
  ignores stale resolutions). Not worth an AbortController here.
- **TagList `(undefined)` if a company lacks `batch`** (account.tsx) / **Me fields typed as
  required** (types.ts:9) — low-likelihood given live `yc me --json` always includes
  id/name/batch; left as-is. If a company ever lacks batch, the tag shows "(undefined)".
