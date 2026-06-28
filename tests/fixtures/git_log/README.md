# git_log fixtures

Mock output of `git log --numstat --oneline` for a fictional Rust project (`sample-app`).
Each file covers one scenario or edge case relevant to watu's Phase 1 parser.

Format: exactly what `git log --numstat --oneline -20` emits — 7-char sha + subject, then
tab-separated `added\tdeleted\tpath` lines, then a blank line between commits.

| File | Scenario |
|---|---|
| `00_empty_history.txt` | empty output — brand new repo, no commits |
| `01_small_edit_small_file.txt` | 3 lines added, 1 deleted in an ~80-line file |
| `02_large_edit_large_file.txt` | 145 added, 80 deleted in a ~500-line file |
| `03_new_file_created.txt` | entirely new file — all additions, 0 deletions |
| `04_file_deleted.txt` | file removed — 0 additions, all deletions |
| `05_rename_same_dir.txt` | renamed within same directory, no edits |
| `06_move_to_different_dir.txt` | file moved to a different directory, with edits |
| `07_directory_renamed.txt` | flat rename of a whole directory (3 files) |
| `08_deep_subtree_moved.txt` | multi-level subtree relocated using `{old => new}` syntax |
| `09_binary_file_changed.txt` | binary files — appear as `- - path` in numstat |
| `10_add_and_remove_directories.txt` | 3 files added (new dir created), 3 files deleted (dir emptied) |
| `11_mixed_realistic.txt` | realistic multi-file commit: edits + new file + config + docs |
| `12_same_file_multiple_commits.txt` | same file appears across 4 commits — aggregation edge case |
| `13_root_level_and_deep_nesting.txt` | root-level files (no dir prefix) + 4-level deep config files |

## Known ambiguities in `--numstat` alone

- `0 0 path` — could be: empty file created, mode/permission change, or submodule pointer update
- `N 0 path` — could be: new file, or existing file with N lines added and 0 deleted
- `0 N path` — could be: file deleted, or file truncated to empty
- Binary files (`- -`) cannot provide diff bar data; watu should render a placeholder bar

To resolve these, combine with `--name-status` (shows A/M/D/R/C per file).
