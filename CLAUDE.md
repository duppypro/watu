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

## Inherits

`AGENTS.md` names the two files above this one: global persona, then domain process.
The development workflow is the `workflow` skill. Load it for a branch, a commit, a pull request, or a merge.

## Issues

Open a GitHub issue before every feature, bug, or tangent.
The standing status issue is https://github.com/duppypro/watu/issues/9.
Refresh it on every `pr-cleanup`.

## Branch policy

The live check is `repo-gate`. The declared policy is the `workflow` skill.
This repo is on that policy (ruleset `protect-default-branch-owner-only`).

## Ignore file

The baseline is the bullet *Write `.gitignore` before the first commit* in the `workflow` skill, § *Branches and worktrees*.
This repo adds `/target/` for Rust build output.
