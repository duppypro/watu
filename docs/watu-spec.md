# Watu — Spec Draft
> Status: brainstorm/scratch — lives in `tmp/`, not committed. Move to a new repo's `docs/` at Step 2.

---

## Tagline

> **"I only watch."** *(and sometimes I don't.)*

The tagline is a direct reference to Uatu the Watcher's oath and its famous violations — establishing from the first line that the tool is observational by default but has intentional intervention hooks (triggered explicitly, never automatic). See §Name & Lore.

---

## Name & Lore

**Name:** `watu`

**Origin:** Derived from *Uatu*, Marvel's "The Watcher" — a cosmic observer bound by oath to observe all events without interfering, who has broken that oath many times when stakes were high enough.

**Why not `uatu` directly:**  
Character names are not copyrightable but *may* be trademarked. Marvel/Disney holds broad trademark registrations. For an open-source CLI tool with no commercial revenue, direct enforcement risk is very low (the open-source community ships Marvel/DC/Star Wars named tools constantly without incident). However, if `watu` ever gains a paid tier, SaaS wrapper, or commercial marketing, "uatu" creates unnecessary legal surface area. `watu` sidesteps this while preserving the phonetic and conceptual identity.

**`watu` as an independent word:**  
`watu` is the Swahili/Bantu word for *"people"* — a common noun in a major language family. This gives it independent standing: a rights holder challenging `watu` would be contesting a real word in another language, which is a much harder case.

**Namespace collision check (2026-06-20):**
| Registry | Collision? | Notes |
|---|---|---|
| GitHub repos | ⚠️ Partial | `wp-plugins/watu`, `leeci/watu`, `asajin/watu` all exist — a decade-old WordPress quiz plugin, completely different ecosystem (PHP/WordPress). No confusion risk. |
| npm | ✅ Clean | No `watu` package found. |
| crates.io | ✅ Clean | Verified 2026-06-20: `crates.io/crates/watu` returns "Crate not found". |
| PyPI | ✅ Clean | No `watu` package found. |
| CLI tool / TUI | ✅ Clean | No existing CLI or TUI tool named `watu`. |

**The broken oath as product philosophy:**  
Uatu broke his oath to "only watch" when the stakes justified it. `watu` is structurally read-only and non-interventionist, but exposes a small set of **explicitly triggered intervention hooks** (future features — e.g., "flag this session for review", "annotate this file with a watu comment", "pause session replay"). These hooks are never automatic; they must be user-triggered. The name signals this tension intentionally.

---

## Problem & Goal

Claude Code harness sessions are opaque. You know something is happening (files changing, tools firing, subagents spawning) but have no live visual of *what*, *where*, *who*, or *how much*. Existing tools (claude-monitor, claude-view) are web dashboards — heavy, browser-dependent, not composable in a terminal workflow. Ralph TUI shows agent hierarchy but is agent-centric, not artifact/file-centric.

**Goal:** A Rust + ratatui TUI that shows, in real time, which files are being touched and by which tool/agent — rendered as an auto-expanding/collapsing directory tree with animated emoji and proportional diff bars. Headless-VPS-friendly (no browser, no localhost).

---

## Differentiation from Existing Tools

| Tool | Type | Closest overlap | Our gap |
|---|---|---|---|
| **Ralph TUI** | TUI | Agent sub-call hierarchy | Agent-centric tree, not file/artifact tree |
| **claude-monitor** | Web dashboard | Session cost + tool call log | Browser-dep, no animated tree, no diff bars |
| **claude-view** | Web client | Tool-call cards (read/edit/bash) | Browser-dep, card-list not tree |
| **agent-deck** | TUI | Multi-agent session manager | Session management, not file activity |
| **ccusage** | CLI | Token cost tracking | No file tree, no live activity |
| **broot** | TUI | File tree navigation | Static snapshot, not activity-reactive |

`watu` is the only file-artifact-centric, animated, diff-bar TUI in this space. Internal shorthand: *"Ralph TUI but file-tree-centric instead of agent-centric."*

---

## Architecture Overview

```
[Phase 1 MVP]
  git refs watcher (inotify on .git/refs/heads/<branch>)
    └─> git log --numstat parser
          └─> AppState (tree + diff counts)
                └─> ratatui render loop

[Phase 2: add .jsonl]
  .jsonl tail task  ──────────────────────────┐
  git refs watcher                            ├─> event merger ─> AppState ─> render
                                              │
[Phase 3: add inotify on working tree]        │
  inotify watcher (working tree) ─────────────┘
```

Three async `tokio::spawn` tasks communicating over `tokio::sync::mpsc` channels, merging into shared `AppState`, rendered each tick (~60ms).

**Phase 3 event correlation logic:**
- inotify write event on path X → read before+after content → line-diff → update diff bar
- `.jsonl` event with `toolName`/`file_path` → associate metadata (tool, agent, line ranges) → queue emoji animation
- inotify write event with NO matching `.jsonl` entry within ~200ms window → "external write" → assign external-write emoji (see §Emoji Map)

---

## TUI Design

### Layout (single session)

```
watu  [session: spec/roguesavvy-draft]  [agent: claude-sonnet-4-6]  [↑↓ to scroll]

 ~/git-projects/rogue-savvy/
 ├── backend/
 │   ├── rogue_savvy/telemetry.py    ✏️·····>  ████████░░░░░  +42 -7  _110
 │   └── rogue_savvy/models.py       👁️·····>  ░░░░░░░░░░░░░  (read L1–88)
 └── docs/
     └── spec-foundation.md          📝·····>  ██░░░░░░░░░░░  +12 -0  _204
```

### Tree behavior
- **Expand on activity:** when any file inside a dir becomes active, all ancestor dirs expand to show it.
- **Collapse on inactivity:** configurable timeout (default 8s) — if no activity in a subtree, it collapses. Dirs with no recent activity are hidden entirely.
- **Only active files shown** — watu is not a file browser; the tree is a live activity feed.
- Unicode box glyphs: `├─` `└─` `│` `┬`. No fallback for MVP (Ghostty assumed).

### Diff bar
- Characters: `█` green (added lines), `▓` red (deleted), `░` gray (unchanged context)
- Phase 1 (git-log): counts from `git log --numstat` — bar shows total added/deleted per last commit touching the file
- Phase 3 (inotify): live update on each write, not just per-commit
- **Width scaling:** `bar_width = (file_line_count / max_sibling_line_count) * available_col_width`  
  Available width = `terminal_width - len(file_path_prefix) - len(emoji_col) - len(count_label)`
- Recomputed on `SIGWINCH` / terminal resize event

### Emoji tool map

| Emoji | Tool / Source | Phase available |
|---|---|---|
| 👁️ | Read | Phase 2 (.jsonl) |
| ✏️ | Edit | Phase 2 |
| 📝 | Write (full file) | Phase 2 |
| 💻 | Bash | Phase 2 |
| 🔍 | Grep / search | Phase 2 |
| 🤖 | Unknown/unrecognized agent tool | Phase 2 |
| 👤 | External write — **default** (most likely: human editor) | Phase 3 |
| 👽 | External write — unknown foreign process | Phase 3 (configurable) |
| 🔧 | External write — mechanical automation (build, make, CI) | Phase 3 (configurable) |
| 🟡 | Git commit (Phase 1 source) | Phase 1 |

**Agent/session emoji:** agent name → stable emoji derived from `name.hash() % 20` over a curated set of 20 non-ambiguous glyphs. Same agent always gets the same emoji within a session; shown in the session header.

**External write emoji selection:** configurable in `watu.toml`. Default: `👤`. Rationale: per-heuristic auto-selection (burst pattern → 🔧, git index lock → 👽, single edit → 👤) is a post-MVP enhancement.

### Emoji animation
- Each tick, emoji position linearly interpolates from source cell (session header row, agent label column) toward target file row in the tree.
- Speed: configurable ticks-per-cell, default reaches target in ~400ms.
- Implementation: no sprite system — simply track `(src_row, src_col, dst_row, dst_col, progress: f32)` in AppState; render emoji glyph at `lerp(src, dst, progress)` each frame.
- Emoji vanishes on arrival. Multiple in-flight animations supported (Vec of animation entries).

---

## Platform Compatibility

**MVP target:** Ghostty terminal emulator, Linux + macOS only.  
**TTY capability detection + graceful fallback** (no-color, ASCII box-drawing, no emoji) is a post-MVP feature. Detect via `$COLORTERM`, `$TERM`, Unicode width probing.

| Crate | Linux | macOS | WSL2 | Win GitBash | Win PS |
|---|---|---|---|---|---|
| `ratatui` + `crossterm` | ✓ | ✓ | ✓ | ⚠️ partial | ✓ (Win Terminal) |
| `notify` | ✓ inotify | ✓ FSEvents | ✓ inotify | ✓ ReadDirChanges | ✓ |
| `similar` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `tokio` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `serde_json` | ✓ | ✓ | ✓ | ✓ | ✓ |

| Terminal | Truecolor | Emoji | Box-drawing |
|---|---|---|---|
| **Ghostty** | ✓ | ✓ | ✓ — **MVP target** |
| Kitty | ✓ | ✓ | ✓ |
| iTerm2 (macOS) | ✓ | ✓ | ✓ |
| Alacritty | ✓ | ⚠️ font-dep | ✓ |
| tmux (wrapper) | ✓ w/ passthrough | ⚠️ needs `allow-passthrough on` | ✓ |
| Windows Terminal | ✓ | ✓ | ✓ — post-MVP |
| xterm / basic | ✗ | ✗ | ✓ — future fallback |

---

## Make vs Buy — Crate Selection

| Crate | Purpose | Alternatives considered | Decision & reason |
|---|---|---|---|
| `ratatui` | TUI rendering, layout, box-drawing, color spans | `cursive`, `tui` (deprecated predecessor) | `ratatui` — active fork of tui-rs, de-facto standard, excellent docs, used by gitui/lazygit ecosystem |
| `crossterm` | Terminal I/O backend for ratatui | `termion` (Linux/macOS only) | `crossterm` — cross-platform, ratatui's recommended backend |
| `notify` | Filesystem event abstraction (inotify/FSEvents) | Raw `inotify` crate (Linux-only), `watchexec` (heavier, subprocess-oriented) | `notify` — cross-platform, async-ready, minimal, exactly scoped |
| `similar` | Line diff (Myers + Patience algorithms) | `diffy` (fewer algorithms), `diff` (older, less maintained) | `similar` — best maintained Rust diff lib, inline change spans, configurable algorithm |
| `tokio` | Async runtime for multi-task architecture | `async-std` | `tokio` — ecosystem standard, best `notify` + `crossterm` integration |
| `serde` + `serde_json` | `.jsonl` line parsing | `simd-json` (faster), `jq`-style libs | `serde_json` — correctness over speed; jsonl throughput is not the bottleneck here |
| `toml` | Config file (`watu.toml`) | `serde_yaml`, CLI-flags-only | `toml` — Rust-native config format, simple for users |

**Rejected: tree-sitter.** Originally considered for syntax-aware AST diff. Rejected because: (a) requires a grammar per language, (b) no-ops on JSON/Markdown/config/binary files, (c) `similar` + inotify covers 100% of the file-change detection use case, (d) tree-sitter is a post-MVP stretch goal if syntax-aware function-boundary highlighting becomes wanted.

---

## Phase 1 MVP — Git-Log Mode

**Data source:** `git log --numstat --oneline -20` on the current branch.  
**Trigger:** inotify watch on `.git/refs/heads/<current-branch>` — fires on every new commit.  
**Fallback:** poll every 5s if inotify on `.git/` is unavailable.

**What you get:**
- File tree of recently-changed files (last N commits, configurable)
- Diff bars from numstat (added/deleted line counts per file per commit)
- Auto-expand/collapse on commit activity
- 🟡 emoji on changed files (no agent metadata yet)
- Replay mode (see §Replay Mode)

**What you don't get yet:**
- Read/grep events (no .jsonl)
- Sub-commit granularity (only see changes at commit time)
- External write detection (no working-tree inotify)
- Emoji animation (no tool metadata to drive source/dest)

---

## Phase 2 — Add `.jsonl` Live Stream

**Data source:** tail `~/.claude/projects/<project-hash>/sessions/<session-id>.jsonl`

**Session `.jsonl` format** — *field names need verification against a real session file before coding Phase 2.* Assumed shape per Claude Code docs:
```json
{"type":"tool_use","toolName":"Read","toolInput":{"file_path":"...","start_line":1,"end_line":50},...}
{"type":"tool_result","toolUseId":"...","content":"..."}
```

**What Phase 2 adds:**
- Read/Grep events visible in tree (invisible to git-log and inotify)
- Tool + agent name metadata → correct per-tool emoji
- Line range annotations on read events
- Emoji animation: tool-emoji flies from session header to file row
- Agent/session identification in header

---

## Phase 3 — Add inotify on Working Tree

**What Phase 3 adds:**
- Sub-commit granularity: WIP writes visible before commit
- External write detection (inotify event, no .jsonl match within ~200ms)
- External write emoji (👤 / 👽 / 🔧 — see §Emoji Map)
- Live diff bar update on each write (not just per-commit)
- Before-snapshot: watu must cache file content hash+snapshot on last-known state to compute before/after diff

---

## Replay Mode

**CLI:** `watu replay path/to/session.jsonl [--speed 2x|instant]`

- Reads `.jsonl` lines sequentially, applies original timestamp delta as `tokio::time::sleep`
- `--speed 2x` halves all deltas; `--instant` removes delays (snapshot mode)
- File content diffs in replay: reconstructed from `toolInput` of Edit/Write entries — no live filesystem needed for replay
- Replay mode doubles as the **automated test harness**: run against a fixed `.jsonl` fixture, assert expected tree events and diff bar values in a snapshot test

---

## Future Features (Post-MVP Backlog)

### Token cost overlay (`$cash$` mode)
- Source: `.jsonl` contains input/output token counts per tool invocation; multiply by current Anthropic pricing per model
- Per-file cost annotation: "editing this file has cost $0.023 this session"
- Session total in header bar
- Historical: cost-per-file over past N sessions (requires persisting session summaries)

### Entropy + complexity correlation
- Per-file: cyclomatic complexity / LOC tracked over commits via `tokei` or `scc` integration
- Correlate with: token spend per file, zero-shot iteration count (commits between "Code Draft" and "Code Approved" per issue)
- Hypothesis: high-entropy/high-complexity files → more agent iterations → higher cost
- Visualize: scatter plot pane (ratatui canvas widget), file complexity on X, token spend on Y, bubble size = iteration count

### Burndown chart pane
- **Incoming rate:** new issues opened (bugs + spec changes) per day — source: GitHub Issues API
- **Close rate:** issues closed per day
- **Projection:** linear/LOWESS fit → predicted milestone date with confidence band
- Renders as a sparkline or canvas chart in a split pane

### Multi-session wrapper (separate tool/layer)
- A meta-TUI that discovers all active sessions under `~/.claude/projects/` and spawns one `watu` pane per session
- Deliberately kept as a separate layer — `watu` single-session mode stays clean and composable
- Could use tmux/zellij splits or ratatui multi-pane layout

### Intervention hooks ("breaking the oath")
- Opt-in, explicitly triggered (never automatic) — honors the Uatu broken-oath narrative
- Candidates: `[a]nnotate` — attach a watu comment to a file's session record; `[f]lag` — mark session for post-hoc review; `[p]ause` replay; `[s]napshot` — write current tree state to JSON for diffing
- These are the only features where watu writes anything

---

## Definition of Done (Step 4 Eval)

1. `watu .` launched during a live Claude Code session: editing a file shows diff bar update within one git commit or (Phase 3) within one write event.
2. `watu replay tests/fixtures/sample-session.jsonl` produces deterministic output matchable against a snapshot assertion.
3. A subdirectory with no activity for 8s collapses; expands again on next activity.
4. Terminal resize (`SIGWINCH`) → diff bar widths recompute correctly within one render tick.
5. Phase 2: reading a file shows 👁️ emoji animating from session header to that file's row and arriving within ~400ms wall-clock.
6. Phase 3: manually editing a file in vim while watu is running shows 👤 external-write emoji on that file.

---

---

## Footnote: "Announcer" — A Separate Tool Concept

*Captured here because it arose during watu naming and is worth preserving. NOT part of watu scope.*

During brainstorming, "Announcer" was considered as a name for `watu`. It was rejected for `watu` but surfaces a distinct and more ambitious idea:

**The concept:** A smart AI-driven narration/commentary layer that sits *between the user and any tool*, not just file-watching tools. Like a sportscaster overlay on top of any CLI tool output, or a dungeon-crawl announcer narrating every move. Two modes:

- **Onboarding mode:** teaches the user what the tool is doing as they use it — live contextual explanation running alongside tool output. Makes any unfamiliar tool immediately approachable.
- **Expert mode:** hides real insights inside whimsical, entertaining commentary. Users choose to run tools *with* the announcer because it's more fun than without, even when they're experts who don't need teaching. The goal: make users unwilling to run any tool without it.

**Name candidates:**
- `announcer` — clear, DCC-native reference (the Announcers who narrate the dungeon crawl for billions of viewers; also universal in sports/games)
- `peanut gallery` — the audience heckling from the cheap seats; implies irreverent commentary on the proceedings
- `heckler` — the most provocative framing; implies it will challenge, question, or mock what it's watching — closest to "hidden true insights wrapped in sarcasm"

**Why it's separate from watu:**
- `watu` is file-artifact-centric and passive; Announcer is tool-agnostic and generative
- Announcer requires an LLM call layer; `watu` is deterministic/structural
- Announcer belongs between stdin/stdout of any tool; `watu` is a standalone TUI process
- They could be composed: `watu | announcer` or Announcer could wrap watu as one of its target tools

**Status:** concept only — no spec, no code, no repo. Record for future Spec Draft.

Sources:
- [GitHub — wp-plugins/watu (WordPress quiz plugin)](https://github.com/wp-plugins/watu/)
- [Ralph TUI: Mission Control for AI Agents](https://www.verdent.ai/guides/ralph-tui-ai-agent-dashboard)
- [agent-deck (GitHub)](https://github.com/asheshgoplani/agent-deck)
- [claude-monitor (GitHub)](https://github.com/szaher/claude-monitor)
