# watu

TUI file-activity observer for Claude Code harness sessions — "I only watch." Public.
**Spec Draft: no code yet, no `Cargo.toml`.** Rust + ratatui when it starts.

## Hard gates

- **No Phase 1 code until #1 and #3 close.** #1 settles config resolution (project → user → flags);
  #3 settles the module boundaries between the event pipeline and the render pipeline.
- **Structurally read-only.** watu never writes to the tree it watches. Intervention hooks
  (annotate, flag, snapshot) are explicit user actions, never automatic.
- **Don't fork wtft's `.jsonl` parser** (#2). Phase 2 reads the same session logs wtft does, so
  two parsers would drift apart.

## Read first

- `docs/watu-spec.md` — the spec: phases, tool→emoji map, timing modes, diff bars.
- `docs/watu-glossary.md` — vocabulary; read it before naming anything.
- `tests/fixtures/` — `git_log/` and `git_patch/` inputs for Phase 1. `git_log/README.md` says what
  each fixture exercises.

## Relations

- Origin: spun off from btw 2026-06-20 (README; watu#5).
- npm name: `@agentic-arts/watu`, because bare `watu` has been taken since 2022 (#4).
