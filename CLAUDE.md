# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Raycast extension that wraps the [`yc` CLI](https://bookface.ycombinator.com/cli) to expose Bookface (Y Combinator's internal network) inside Raycast. It does not talk to any HTTP API directly — every data fetch shells out to the `yc` binary with `--json` and parses stdout.

Three commands, all `view` mode, all macOS-only:

- `search` (Search YC) — `yc search <query> --json`, type-filtered list with detail sidebar
- `ask` (Ask YC) — `yc agent <question> --json`, Form → Detail flow
- `account` (YC Account) — `yc me --json`

## Commands

```bash
npm run dev          # ray develop — hot-reload in Raycast
npm run build        # ray build — production build
npm run lint         # ray lint
npm run fix-lint     # ray lint --fix
npm run publish      # publish to Raycast Store (do not run unprompted)
```

There is no test framework configured. `ray develop` reloads the extension live in Raycast; that is the dev loop.

## Architecture

### CLI integration (`src/lib/yc.ts`)

This is the only place that executes the `yc` binary. Two patterns:

1. **`runYc<T>(args)`** — `execFile`-based, returns a `YcResult<T>` discriminated union: `{ ok: true, data }` or `{ ok: false, kind: "missing-cli" | "not-authed" | "error", message }`. Used by `ask` and `account`. Callers branch on `kind` to render the right empty state.
2. **`useExec` from `@raycast/utils`** — used in `search` for live, debounced search-as-you-type. Takes the resolved binary path and parses stdout in `parseOutput`.

`resolveYcPath()` checks the user's `ycPath` preference, then falls back to `~/.local/bin/yc`, Homebrew paths, and `ycp` variants. Result is cached in module scope until `ENOENT` invalidates it. Always call this before invoking the CLI.

Three failure modes are user-visible and must be preserved when adding new CLI calls:

- **missing-cli** → show `INSTALL_COMMAND` with copy-to-clipboard action
- **not-authed** → show `LOGIN_COMMAND` (detected via stderr/stdout sentinels in `NOT_AUTHED_SENTINELS`)
- **error** → surface message, copy-to-clipboard action

### Search rendering (`src/lib/items.tsx`)

`renderItem({ item, isShowingDetail, toggleDetail })` switches on `SearchItem["type"]` (8 variants: `user`, `yc_company`, `non_yc_company`, `school`, `post`, `deal`, `employer`, `startup_library`). Each type has its own renderer that returns a `<List.Item>` with type-specific accessories, detail markdown, and ActionPanel.

When adding a new search item type:

1. Add the variant to `SearchItem` and the type-specific `*Attributes` interface in `src/lib/types.ts`.
2. Add label to `SEARCH_TYPE_LABELS` and position in `SEARCH_TYPE_ORDER` (controls dropdown order and result sort).
3. Add a `case` in `renderItem` and a `render*` function.
4. Use `UniversalActions` for the standard Open/Toggle Sidebar/Copy URL/Copy as Markdown set so shortcuts stay consistent across types.

Real `yc search` JSON fixtures live in `.github/docs/yc-search-*.json` — useful when adding renderers for unfamiliar item types or debugging shape mismatches.

### Recent searches hook (`src/hooks/use-recent-searches.ts`)

Generic `LocalStorage`-backed history. Both `search` and `ask` consume it with different storage keys (`search-recents`, `ask-recents`) and different limits.

`search` only persists a query after the search succeeds (guarded by `lastPersistedRef` to avoid double-writes when results re-render).

## Conventions

- Detail markdown is built by `*Markdown(attrs)` helpers; the same string is reused for sidebar detail and `MarkdownPreview` push views (the preview strips the leading `# Title` since it's set as `navigationTitle`).
- For error toasts use `showFailureToast(error, { title })` from `@raycast/utils` — it ships a "Copy Logs" action (private/dev builds) or "Report Error" action (published store builds) by default, so we don't roll our own.
- Keep the `yc` CLI as the single source of truth for data. Do not add direct HTTP calls to Bookface.
- 350ms debounce on search input; `keepPreviousData: true` so the list doesn't flash empty between queries.
- `subagent_type`-style discriminated unions (`YcResult`, `SearchItem`) are the preferred pattern for fallible/variant data — branch on `kind`/`type`, never null-check.
