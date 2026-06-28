# Watu — Domain Glossary

Canonical vocabulary for humans, AI agents, and code. When in doubt about what to call something — in conversation, in a commit message, in a variable name — use this document.

**Entry format:**
- **Plain language** — what to say when talking or writing about it
- `code_name` — Rust identifier(s); snake_case fields/vars, PascalCase types, SCREAMING_SNAKE consts
- *config key* — `watu.toml` key where applicable
- **Phase** — when this concept first becomes relevant (1, 2, 3, or All)

---

## A

### `active file`
A file currently visible in the watu tree because it has had at least one event since watu started, and its subtree has not yet timed out.

- `FileEntry` (tree node struct) or `TrackedFile` (candidate name)
- Phase: All

### `agent`
The Claude Code AI running a session. Multiple sub-agents may appear within one session; each gets its own `agent_emoji`.

- `agent_name: String` field in `SessionInfo`
- Phase: Phase 2 (Phase 1 has no agent identity)

### `agent emoji`
A stable emoji assigned to one agent, derived deterministically: `emoji_palette[hash(agent_name) % 20]`. Same agent always gets the same emoji within a session. Shown in the session header; serves as the emoji animation source.

- `fn agent_emoji(name: &str) -> char`
- Phase: Phase 2

### `animation queue`
The ordered list of in-flight `EmojiFlights` and pending `FlashEvents` held in `AppState`. Consumed each render tick. Events queue here when they arrive faster than `bar_flash_ms` (Mode 3).

- `animations: Vec<EmojiFlight>`, `flash_queue: VecDeque<FlashEvent>` in `AppState`
- Phase: All

### `AppState`
The central mutable model of everything watu knows: the file tree, active animations, session info, timing counters, and bar states. Shared across async tasks.

- `struct AppState { ... }` — accessed via `Arc<Mutex<AppState>>`
- Phase: All

### `available_col_width`
The number of terminal columns reserved for drawing diff bars, computed each render tick after subtracting path, emoji-column, and label widths.

- `available_col_width: u16`
- Phase: All

---

## B

### `bar_flash_ms`
How long the diff overlay is shown on screen before snapping back to resting state. A **perception constant** — it never scales with replay speed or BPM.

- `bar_flash_ms: u64`
- *config key*: `bar_flash_ms` (default `400`)
- Phase: All

### `beat`
The fundamental time unit in Mode 2 (beat replay). Each commit occupies one beat. Duration = `60 000 / bpm` ms. At 90 BPM: 666.667 ms.

- see `beat_ms`
- *config key*: `replay_bpm`
- Phase: Mode 2 only

### `beat_ms`
Computed duration of one beat in milliseconds.

- `let beat_ms: f64 = 60_000.0 / bpm;`
- Phase: Mode 2 only

---

## C

### `collapse_timeout_ms`
Inactivity threshold: if no event touches a subtree for this long, the subtree collapses (files hidden, directory folded). Resets on each new event.

- `collapse_timeout_ms: u64`
- *config key*: `collapse_timeout_ms` (default `8000`)
- Phase: All

### `commit depth`
How many commits back `git log` looks on startup and after each trigger.

- `commit_depth: u32`
- *config key*: `commit_depth` (default `20`)
- Phase: Phase 1

### `commit event`
A new git commit detected via inotify on `.git/refs/heads/<branch>`. The primary event type in Phase 1. Triggers a diff bar flash for all changed files.

- `WatuEvent::Commit(CommitInfo { sha: String, files: Vec<FileDiff> })`
- Phase: Phase 1+

### `content snapshot`
watu's cached copy of a file's content at the last known state. Used in Phase 3 to compute before/after diffs for WIP writes.

- `struct ContentSnapshot { path: PathBuf, content: String, line_count: u32 }`
- Phase: Phase 3

---

## D

### `diff bar`
The horizontal bar displayed next to each active file in the tree. In **resting state**: plain `░` fill proportional to `file_line_count`. During a **flash**: positional diff overlay (`░▓█` pattern).

- `fn render_diff_bar(entry: &FileEntry, width: u16, now: Instant) -> Span`
- Phase: All

### `diff overlay`
The `░▓▓░░████` content shown inside a diff bar during a flash. Synonym: **flash state**. Shows unchanged (`░`), deleted (`▓`), and added (`█`) lines at their hunk positions. Lasts `bar_flash_ms`.

- `BarState::Flash { diff: FileDiff, started_at: Instant }`
- Phase: All

### `display_gap_ms`
In Mode 1 replay, the computed real-time wait between displaying two consecutive commits.

```
display_gap_ms = min(real_gap_ms / speed, ffw_idle_cap_ms)
```

- `let display_gap_ms: f64 = (real_gap_ms as f64 / speed).min(ffw_idle_cap_ms as f64);`
- Phase: Mode 1 replay

---

## E

### `emoji animation` / `emoji flight`
The visual effect of a tool emoji flying from the session header row to the target file's row in the tree. Departs at flash-start; arrives at flash-end (`emoji_speed_ms`).

- `struct EmojiFlight { emoji: char, src: (u16, u16), dst: (u16, u16), progress: f32 }`
- *config key*: `emoji_speed_ms` (default `400`)
- Phase: Phase 2+

### `emoji palette`
The curated set of 20 non-ambiguous emoji used for agent identity hashing. Fixed at compile time.

- `const EMOJI_PALETTE: [char; 20] = [...]`
- Phase: Phase 2

### `event`
Any incoming notification that drives watu's animation pipeline. Three subtypes:

| Subtype | Source | Phase |
|---|---|---|
| `commit event` | inotify on `.git/refs/` | 1+ |
| `tool event` | new `.jsonl` line | 2+ |
| `fs write event` | inotify on working tree | 3 |

- `enum WatuEvent { Commit(CommitInfo), Tool(ToolEvent), FsWrite(FsWriteEvent) }`
- Phase: All

### `external write`
An `fs write event` that has no matching `.jsonl` tool event within the correlation window (~200 ms). Presumed to be a human editor, build tool, or foreign process.

- `WriteSource::External` variant; emoji = `external_write_emoji` config
- *config key*: `external_write_emoji` (default `"👤"`)
- Phase: Phase 3

---

## F

### `FFW_IDLE` (fast-forward idle)
The behavior in Mode 1 where large gaps between commits are compressed to `ffw_idle_cap_ms`. Named by analogy with video fast-forward through silent/idle stretches.

- see `ffw_idle_cap_ms`
- Phase: Mode 1 replay

### `ffw_idle_cap_ms`
The ceiling on `display_gap_ms` in Mode 1 replay. Any gap that would display longer than this is capped here, preventing watu from stalling on overnight or weekend commit gaps.

- `ffw_idle_cap_ms: u64`
- *config key*: `ffw_idle_cap_ms` (default `5000`)
- Phase: Mode 1 replay

### `file_line_count`
The total number of lines in a file at its current (post-event) state. Drives resting bar width. Sourced from `FileDiff.new_line_count` after each event, or cached in the `ContentSnapshot`.

- `file_line_count: u32` (field in `FileEntry`)
- Phase: All

### `FileDiff`
The complete positional diff for one file in one event: old/new paths, line counts, list of hunks, binary flag.

```rust
struct FileDiff {
    old_path:       String,
    new_path:       String,    // == old_path unless renamed/moved
    old_line_count: u32,       // drives resting bar width before flash
    new_line_count: u32,       // drives resting bar width after flash
    hunks:          Vec<Hunk>,
    is_binary:      bool,      // true → render placeholder ░ bar, no hunk data
}
```

Built from `@@` hunk headers in `git log --patch` (Phase 1), or from `old_string`/`new_string` in `.jsonl` Edit events (Phase 2).

- Phase: All

### `flash`
The `bar_flash_ms`-duration window during which the diff bar shows the diff overlay. Begins when an event arrives; ends when `bar_flash_ms` expires, at which point the bar snaps back to resting state with the new `file_line_count`.

- `BarState::Flash` → `BarState::Resting` on expiry
- Phase: All

### `fs write event`
An inotify notification that a file in the working tree was written. May be a `tool event` (if matched to `.jsonl`) or an `external write` (if unmatched within ~200 ms).

- `WatuEvent::FsWrite(FsWriteEvent { path: PathBuf, source: WriteSource, timestamp: Instant })`
- Phase: Phase 3

---

## H

### `hunk`
A contiguous region of change within a file — the basic unit of a positional diff. Defined by its start line and changed-line counts in the old and new file.

```rust
struct Hunk {
    old_start: u32,   // 1-indexed; line in old file where hunk begins
    old_count: u32,   // lines deleted from old file
    new_start: u32,   // 1-indexed; line in new file where hunk begins
    new_count: u32,   // lines added to new file
}
```

Sourced from `@@` headers in `git log --patch --unified=0` (Phase 1) or derived by locating `old_string` in the file for `.jsonl` Edit events (Phase 2).

- Phase: All

---

## I

### `intervention hook`
A user-triggered action that breaks watu's observer-only stance — annotating a file, flagging a session, or snapshotting state. Inspired by Uatu's oath-violations. Never automatic.

- `enum HookAction { Annotate, Flag, Snapshot, PauseReplay }` (post-MVP)
- Phase: Post-MVP

---

## M

### `max_sibling_line_count`
The largest `file_line_count` among files at the same directory level. Used as the proportional-width denominator so the largest sibling fills `available_col_width` and all others scale relative to it.

- `max_sibling_line_count: u32` (computed per directory node during render)
- Phase: All

### `Mode 1` — timestamp-proportional replay
Replay that respects original wall-clock gaps between commits, scaled by `replay_speed` and capped by `ffw_idle_cap_ms`.

- `ReplayMode::Timestamped { speed: f64, ffw_idle_cap_ms: u64 }`
- CLI: `watu replay [--speed <Nx>] [--ffw-idle-cap <ms>] <file>`

### `Mode 2` — beat replay
Replay that ignores timestamps and assigns each commit exactly one beat (`max(beat_ms, bar_flash_ms)` total).

- `ReplayMode::Beat { bpm: f32 }`
- CLI: `watu replay --bpm <N> <file>`

### `Mode 3` — live watch (default)
The primary production mode. Events drive animation as they arrive in real time. Events queue if they arrive faster than `bar_flash_ms`.

- `RunMode::Watch`
- CLI: `watu [watch] [<dir>]`

---

## P

### `per_commit_ms`
In Mode 2, the total display time per commit: `max(beat_ms, bar_flash_ms)`. Guarantees at least one full flash even at high BPM.

- `let per_commit_ms: f64 = beat_ms.max(bar_flash_ms as f64);`
- Phase: Mode 2

---

## R

### `resting state`
The steady-state appearance of a diff bar between events: a plain bar of `░` characters, width proportional to current `file_line_count`. No `▓` or `█` markers at rest.

- `BarState::Resting { line_count: u32 }`
- Phase: All

### `replay_bpm`
Beat tempo for Mode 2 replay. 90 BPM → 666.667 ms per beat.

- `replay_bpm: f32`
- *config key*: `replay_bpm` (default `90.0`)

### `replay_speed`
Speed multiplier for Mode 1 replay. `"10x"` means gaps are displayed 10× faster.

- `replay_speed: f64` (parsed from config string `"10x"`)
- *config key*: `replay_speed` (default `"1x"`)

---

## S

### `session`
One continuous Claude Code agent run, identified by session ID. Produces one `.jsonl` file at `~/.claude/projects/<project-hash>/sessions/<session-id>.jsonl`.

- `struct SessionInfo { id: String, agent_name: String, project_path: PathBuf }`
- Phase: Phase 2+

### `session header`
The top line of the watu TUI, showing session name, agent emoji, model name, and scroll hint. This row is the **source cell** for all emoji animations.

- `fn render_header(state: &AppState, width: u16) -> Line`
- Phase: Phase 2+ (Phase 1 shows watch_dir and branch only)

### `static_hold_ms`
In Mode 2, the time between flash-end and the next commit's flash-start: `per_commit_ms - bar_flash_ms`. Zero when `bar_flash_ms >= beat_ms`.

- `let static_hold_ms: f64 = per_commit_ms - bar_flash_ms as f64;`
- Phase: Mode 2

---

## T

### `tool event`
A single tool invocation in a Claude Code session, represented as one `tool_use` line in the `.jsonl` log. Each tool event can trigger a diff bar flash and emoji animation.

- `struct ToolEvent { tool: ToolKind, file_path: Option<PathBuf>, hunk: Option<Hunk>, agent: String, timestamp_ms: f64 }`
- Phase: Phase 2+

### `ToolKind`
Enum of recognized Claude Code tool types, each mapped to a display emoji.

```rust
enum ToolKind { Read, Edit, Write, Bash, Grep, Unknown }
```

| Variant | Emoji | Notes |
|---|---|---|
| `Read` | 👁️ | Shows read range annotation, no diff bar flash |
| `Edit` | ✏️ | Positional diff from `old_string`/`new_string` |
| `Write` | 📝 | Full-file replace; diff against `ContentSnapshot` |
| `Bash` | 💻 | File path optional (not all Bash touches files) |
| `Grep` | 🔍 | No content change |
| `Unknown` | 🤖 | Unrecognized `toolName` in `.jsonl` |

- Phase: Phase 2+

---

## V

### `visual_total`
The "virtual" number of lines a diff bar represents during the flash state — old unchanged lines plus all newly added lines. May temporarily exceed both `old_line_count` and `new_line_count`.

```
visual_total = old_line_count + Σ hunk.new_count
scale        = available_col_width / visual_total
```

- `let visual_total: u32 = diff.old_line_count + diff.hunks.iter().map(|h| h.new_count).sum::<u32>();`
- Phase: All

---

## W

### `watch_dir`
The filesystem path watu is observing. Defaults to the current working directory.

- `watch_dir: PathBuf`
- *config key*: `watch_dir` (default `.`)
- Phase: All

### `WIP write` (work-in-progress write)
A file write detected by inotify in Phase 3 that has not yet been committed to git. watu shows these as sub-commit-granularity events, flashing the diff bar against the last `ContentSnapshot`.

- `FsWriteEvent` with `source: WriteSource::Agent` or `WriteSource::External`
- Phase: Phase 3

### `working tree`
The actual project files on disk as checked out — distinct from git history or the index. Phase 3 watches the working tree directly via inotify for WIP writes.

- `watch_dir: PathBuf` (the root being watched)
- Phase: Phase 3 (Phase 1 uses it indirectly via `git log`)

---

## Display Notation Reference

| Symbol | Name | Meaning |
|---|---|---|
| `░` | neutral fill | Resting state (whole bar), or unchanged lines during flash |
| `▓` | deletion marker | Deleted lines during flash; positioned at their location in the old file |
| `█` | addition marker | Added lines during flash; inserted at hunk position; may widen bar |
| `+N -M` | count label | Total lines added / deleted; derived from `Σ hunk.new_count` and `Σ hunk.old_count` |
| `_N` | line-count label | Current `file_line_count` (e.g. `_110` = 110 lines); `_` prefix is a visual separator |
| `✏️·····>` | emoji in flight | Tool emoji animating toward target file row; persists for `emoji_speed_ms` |
| `(read L1–88)` | read annotation | Shown for 👁️ Read events instead of a diff bar; reads have no content change so no flash is triggered |
