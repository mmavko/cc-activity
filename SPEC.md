# Claude Code Session Activity Tracker — Spec

A GitHub-contributions-style heatmap of Claude Code usage, computed entirely
client-side from local session files. No backend, no telemetry.

## Architecture

- Single self-contained file: `docs/index.html` (inline CSS/JS, no build step,
  no external requests, no dependencies). Lives under `docs/` so it can be
  published directly via GitHub Pages.
- Runs either by opening the file directly (`file://`) or via a local web
  server (`http://localhost`) — both work, no backend required.
- Reads `~/.claude/projects` via the File System Access API
  (`showDirectoryPicker` / `FileSystemDirectoryHandle`). Chromium-only — no
  Firefox/Safari support. The app detects this and shows an "unsupported
  browser" message instead of the picker.
- Persists the picked `FileSystemDirectoryHandle` in IndexedDB
  (`cc-activity` database, `kv` store, key `rootHandle`) so the user doesn't
  have to re-pick the folder on every launch. Re-granting permission on a
  stored handle still requires one user gesture (`requestPermission` can't
  fire silently), so the app always shows a "continue with `<folder>`" button
  rather than skipping straight to data.
- The handle is only written to IndexedDB after a scan of that folder
  actually finds session files. Picking an empty or wrong folder does not
  overwrite a previously-working stored handle, and the gate always offers a
  "choose a different folder" path back to the picker — there's no dead end.
- Known caveat: on `file://`, IndexedDB does not appear to be partitioned per
  file or per directory in Chrome — it behaves as one shared storage bucket
  across local HTML pages in the profile. Not a problem for personal use, but
  the stored handle isn't sandboxed to this specific file the way it would be
  on a real origin. Serving via `localhost` avoids this if it ever matters.

## Data source & file discovery

- Each project lives under `~/.claude/projects/<encoded-cwd>/`, containing
  one `.jsonl` file per session, plus a `<session-uuid>/subagents/*.jsonl`
  subfolder for any subagent transcripts spawned during that session.
- **Message scanning** only globs one level deep: `~/.claude/projects/*/*.jsonl`.
  Subagent transcripts are deliberately excluded — they contain
  orchestrator-to-subagent prompts that look like real human messages (same
  `role: user`, plain string content) but were never typed by a person.
  Restricting the glob depth excludes them for free, no per-line check needed.
- **Disk size** is measured separately and recursively: `dirSizeBytes()` walks
  every file under each project folder at any depth (subagent transcripts,
  shell snapshots, anything else), summing `File.size`. This is the *true*
  on-disk footprint of the project, not just the top-level session files used
  for message counting — the two scopes are intentionally different.
- **Session scratchpads are excluded.** Claude Code gives each session a
  temp directory at `/tmp/claude-<uid>/<encoded-cwd>/<session-uuid>/scratchpad`.
  When an agent starts a session whose cwd is inside one, Claude Code registers
  that temp path as a project in its own right. Those transcripts are real but
  agent-authored, and their directory is disposable, so they're dropped twice
  over: by folder name (`-private-tmp-claude-<digits>-…`) before any parsing,
  and again by `cwd` after parsing, in case the encoded name doesn't match. The
  uid is machine-specific, so the digits are matched as `\d+`, and both `/tmp`
  and `/private/tmp` are accepted.
- The folder name for each project is a dash-encoded absolute path (e.g.
  `-Users-myron-dev-foo`). This encoding is lossy whenever a real path segment
  contains a literal hyphen (e.g. `claude-coding`), so it's not naively
  decoded for display. Instead, the app reads the `cwd` field that's present
  on most JSONL lines and uses that as the authoritative display path,
  falling back to a naive dash→slash decode only if no line in that project
  has a `cwd` field.

## Message filtering

A JSONL line counts as one human-authored message iff **all** of:

1. `type === "user"`
2. `message.role === "user"`
3. `isMeta` is not `true`
4. No top-level `toolUseResult` field, and if `message.content` is an array,
   none of its blocks have `type === "tool_result"`
5. `message.content` flattens to text: if it's already a string, used as-is;
   if it's an array, every block of `type: "text"` is joined (in order) and
   any non-text blocks — images, etc. — are ignored rather than disqualifying
   the message. An image pasted with no caption flattens to an empty string,
   which still counts as one message (it fails none of the exclusion checks
   below); an image with a caption counts using just the caption text.
6. The flattened text, left-trimmed, does not start with any of these
   synthetic wrapper tags: `<command-name>`, `<command-message>`,
   `<command-args>`, `<local-command-caveat>`, `<local-command-stdout>`,
   `<local-command-stderr>`, `<system-reminder>`, `<task-notification>`.
7. The flattened text does not start with `[Request interrupted by user`
   (covers both the bare interrupt and "...for tool use" variants).

Checks 3, 6, and 7 are all required together — no single field reliably flags
every synthetic case. Slash-command artifacts (`<command-name>`,
`<local-command-stdout>`) carry no `isMeta` flag at all; conversely, skill
injections and "Continue from where you left off." carry `isMeta: true` but
no recognizable tag prefix; interrupt markers carry neither.

This relies on undocumented internal conventions of the Claude Code CLI (the
wrapper tags, `isMeta`, the subagent directory layout) and could break
silently on a future CLI version that changes them. There's no public schema
to pin against.

## Day bucketing

Each message is bucketed by **local calendar day** (`date.getFullYear()` /
`getMonth()` / `getDate()`), not the raw UTC date in the timestamp, so a
late-night session lands on the day it felt like it happened on, not
whichever UTC date it happens to serialize to.

## UI

### Gate / loading states

- **First run** (no stored handle): instructions + a "choose
  `~/.claude/projects`" button.
- **Subsequent run** (stored handle found): "continue with `<folder name>`" —
  one click re-requests permission and scans.
- **Scanning**: progress bar showing `done / total` steps, where the steps
  span both the per-session-file parsing pass and the per-project disk-size
  walk.
- **No sessions found** / **scan failed**: explains the problem and always
  resets the button to open the picker again ("choose a different folder") —
  never a dead end.
- **Unsupported browser**: shown if `window.showDirectoryPicker` doesn't
  exist; no picker button offered.
- A "change folder" ghost button appears in the top-right of the topbar once
  data has loaded, for switching to a different folder without clearing
  IndexedDB by hand.
- A refresh icon button appears next to the blinking cursor in the brand
  once data has loaded. It re-requests permission on the already-granted
  handle (kept in memory as `currentRootHandle`, no IndexedDB round-trip
  needed) and re-runs the scan in place — a single click to pick up new
  sessions, instead of reloading the page and clicking "continue".

### Layout

Sidebar (fixed width, independently scrollable) + main panel (independently
scrollable), under a sticky topbar styled like a terminal prompt
(`me@local:~/.claude/projects$ activity`).

### Sidebar

- "All projects" pinned at the top, always present, aggregating every
  project's daily counts.
- One row per project, sorted by most recent activity (descending).
- Each row shows: display name (last two path segments, e.g.
  `.../dev/claude-coding`), total filtered message count, a relative
  last-active label (`today` / `yesterday` / `Nd ago` / `Nw ago` / ISO date)
  with a native tooltip giving the exact date and time, and the project's
  disk size right-aligned on the same line.
- The footnote "runs 100% locally, nothing leaves this browser tab" is
  pinned to the bottom of the sidebar (flex column, not part of the
  scrolling list).

### Main panel

- Heading shows the selected project's display name (or "all projects") and
  a "tracking since `<date>`" subline.
- Five stat blocks: **messages**, **active days**, **current streak** (in
  days, counting backward from today; today itself doesn't break the streak
  if it has no activity yet), **busiest day** (date + count), **disk size**.
- Below that, the heatmap: weeks as columns (Monday at the top, Sunday at the
  bottom), spanning from the selection's first activity (local midnight) to
  today. Range is adaptive per selection, not a fixed trailing year; long
  ranges scroll horizontally inside the heatmap card rather than compressing.
- Shade intensity uses a 5-level scale (plus the empty/zero level) with
  thresholds computed from quantiles of nonzero day-counts across **all
  projects**, not the currently selected project alone. This keeps shading
  comparable across projects and avoids a project with only one or two
  active days always degenerating to the lightest shade.
- Hovering a day with at least one message shows a floating tooltip with the
  full weekday/date and message count. Empty days have no hover handlers at
  all, so no tooltip flicker on zero-count cells.
- Cells are laid out as contiguous 14×14 hit-boxes with inset padding and
  `background-clip: content-box` (rather than flex `gap`), so the visible
  swatch still looks like an 11×11 square with a gap, but the pointer never
  crosses a real gap between hit-areas — eliminates hover flicker when
  sweeping the mouse across the grid.

## Persistence

IndexedDB (`cc-activity` / `kv`):
- `rootHandle` — the granted `FileSystemDirectoryHandle`, written only after
  a successful (non-empty) scan.
- `lastProject` — the last-selected sidebar key (a project's folder name, or
  `'ALL'`). Restored on load; falls back to `'ALL'` if the stored key no
  longer exists among the scanned projects.

## Not implemented

- **No caching of parsed results.** Every load does a full rescan: reads and
  parses every top-level `.jsonl` file's full text, plus a recursive
  metadata-only walk for disk size. Fine at the current data volume
  (tens of MB, well under a second); would need addressing if session
  history grows much larger.
  - Natural approach if needed: cache per-day message counts (and disk size)
    in IndexedDB keyed by each file's size + mtime, and only re-parse files
    whose key changed since the last run.
