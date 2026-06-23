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

### [F6] Large `yc --json` output truncated by Node pipe capture — BLOCKING (fixed)

**Where:** src/lib/yc.ts `runYc` (execFile stdout capture); surfaced in `search` for
big result sets (e.g. "stripe", "fundraising").
**Issue:** `yc search <q> --json` for large results (~141KB) returned **truncated**
JSON to the extension — varying cutoffs at **65479 (~64KB)** and **131003 (~128KB)**
bytes — so `JSON.parse` failed and search showed "Failed to parse yc output".
**Investigation (corrected a wrong initial hypothesis):**
- Initial (WRONG) hypothesis: "YC randomly truncates its JSON." Chris pushed back.
- Redirecting `yc search stripe --json > file.json` at the shell: **6/6 runs complete
  and valid (141120 bytes).** So the CLI output is complete and deterministic — YC is
  NOT truncating.
- Capturing the same command via Node `execFile` (how `runYc` works): truncated at
  64KB/128KB **pipe-buffer boundaries**. Root cause: `yc` exits before its stdout pipe
  fully drains to the parent; Node only receives what cleared the pipe buffer. NOT a
  `maxBuffer` issue (141KB ≪ the 4MB limit; that would throw, not truncate).
**Fix (this session):** capture via a temp file instead of a pipe — run
`sh -c '"$0" search "$1" --json > "$2"'` with the binary/query/path as **positional
args** (`$0`/`$1`/`$2`, never interpolated → no shell injection), then read the file.
The OS completes the write regardless of the child's flush timing. Verified: 3/3 runs
complete, items=10.
**Resolution:** FIXED — temp-file capture in `runYc`.

### [F7] yc server rate-limits rapid calls (HTTP 429) — partially-addressed

**Where:** server-side; amplified by search firing `yc search` per debounced keystroke
plus the mount `yc me` probe.
**Issue:** Rapid repeated `yc search` calls return `Search failed (429): Rate limit
exceeded, check the Retry-After header`. Empty/error output → "no results" or a raw
error dump. Reproduced consistently under fast successive invocations.
**Fix (this session):** (a) longer search debounce to cut call volume; (b) smarter
recent-search capture so we don't fire/persist every partial prefix. A dedicated 429
failure kind with Retry-After handling is **deferred** (see below) — normal typing
shouldn't hit it once debounce + capture are tuned.
**Resolution:** PARTIALLY-ADDRESSED — pressure reduced; explicit 429 kind deferred.

### [F8] Recent searches saved every partial keystroke — non-blocking (fixed)

**Where:** src/search.tsx recent-search persistence (`onData`/`addRecentSearch`).
**Issue:** Every debounced query that returned data was saved, so recents filled with
noise: `fun`, `fundraisi`, `fundraising`, `S`, `Stri`, `Strip`, `Stripe`… (confirmed by
screenshot). A later, longer query that supersedes an earlier prefix should win.
**Fix (this session):** only persist deliberate searches and drop a prior recent that is
a strict prefix of the new query (so `Stripe` replaces `Stri`/`Strip`). See use-recent-searches.
**Resolution:** FIXED.

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
