# watu

> **"I only watch."** *(and sometimes I don't.)*

A Rust + ratatui TUI that gives real-time visual observability of [Claude Code](https://claude.ai/code) harness sessions — which files are being touched, by which tool or agent, shown as an auto-expanding/collapsing directory tree with animated emoji and proportional diff bars.

**Status: Spec Draft** — no production code yet. See [`docs/watu-spec.md`](docs/watu-spec.md) for the full spec.

---

## The Problem

Claude Code harness sessions are opaque. Files change, tools fire, subagents spawn — but there is no live visual of *what*, *where*, *who*, or *how much*. Existing tools (claude-monitor, claude-view) are web dashboards: browser-dependent, not composable in a headless terminal workflow. Ralph TUI shows agent call hierarchy but is agent-centric, not file/artifact-centric.

## What watu does

```
watu  [session: my-project · claude-sonnet-4-6]

 ~/git-projects/my-project/
 ├── src/
 │   ├── main.rs          ✏️·····>  ████████░░░░░  +42 -7  _110
 │   └── lib.rs           👁️·····>  ░░░░░░░░░░░░░  (read L1–88)
 └── docs/
     └── spec.md          📝·····>  ██░░░░░░░░░░░  +12 -0  _204
```

- **Auto-expanding tree:** directories expand when files inside become active; collapse after inactivity (default 8s). Only active files shown — not a file browser, a live activity feed.
- **Diff bars:** proportional green/red/gray bars scaled to file size relative to siblings.
- **Emoji tool map:** each Claude Code tool gets an emoji (👁️ Read, ✏️ Edit, 📝 Write, 💻 Bash, 🔍 Grep). External writes (not from the harness) get 👤 by default.
- **Animated emoji:** tool emoji flies from the agent/session header to the target file row on each invocation.
- **Replay mode:** `watu replay session.jsonl` replays a past session at configurable speed — doubles as the automated test harness.

## Name & Lore

`watu` derives from *Uatu*, Marvel's "The Watcher" — a cosmic observer bound by oath to watch all events without interfering, who has broken that oath many times when stakes were high enough.

`watu` is also the Swahili/Bantu word for *"people"* — an independent real word, not just a derived trademark reference.

The broken-oath narrative is intentional product philosophy: watu is structurally read-only and non-interventionist, but exposes a small set of explicitly triggered **intervention hooks** (annotate, flag, snapshot). These are never automatic. The tagline signals the tension.

## Architecture (3 phases)

| Phase | Data source | What you get |
|---|---|---|
| **1 — MVP** | `git log --numstat` | File tree of recently committed changes, diff bars per commit |
| **2** | `.jsonl` session log tail | Read/grep events, tool metadata, emoji animation, agent identity |
| **3** | `inotify` on working tree | Sub-commit granularity, external write detection (👤👽🔧) |

## Stack

- **Language:** Rust
- **TUI:** `ratatui` + `crossterm`
- **Filesystem watch:** `notify` crate (inotify/FSEvents)
- **Diff:** `similar` crate (Myers algorithm)
- **Async:** `tokio`
- **Target terminals (MVP):** Ghostty, Kitty, iTerm2 — full emoji + truecolor required

## Future Roadmap

- Token cost overlay per file and session (`$cash$` mode)
- Entropy/complexity correlation with token spend and zero-shot iteration count
- Burndown chart pane (incoming bug/spec rate vs close rate → milestone prediction)
- Multi-session wrapper (separate meta-TUI layer)

---

## See Also

`watu` is part of a broader tooling exploration by [Agentic Arts](https://github.com/duppypro). A related concept — a smart AI narration layer that wraps *any* tool with live commentary and hidden insights — is being explored separately under the working name **Announcer** (also: `peanut gallery`, `heckler`). See the footnote in [`docs/watu-spec.md`](docs/watu-spec.md).
