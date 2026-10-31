# qs — Flags Reference

```
qs <command> <file|dir> [flags]
```

## Commands

| Command | Writes files? | Description |
|---|---|---|
| `analyze` | No | Lists split candidates (func/var/type + line span) per file. |
| `preview` | No | Renders the full proposed folder tree + generated file contents. Always run before `split`. |
| `split`   | Yes | Writes the split files, moves the original to `~/.Trash` via `mv` (never `rm`). |
| `blocks`  | No | Report SAFE sub-helper candidates inside a single-function fragment (for manual extraction past the mechanical floor). |
| `extract` | Yes | Lift SAFE ranges into concern-named static helpers; body text moves verbatim. Backs up shell to `~/.Trash/<name>.mb-<HHMMSS>`. |
| `audit-inline` | No | Symmetric reassembly audit: inline fragment `#include`s on current + backup, diff meaningful-line multisets (catches overwritten-fragment data loss). |
| `help`    | No | Prints this reference. |

## Path argument

- **File**: operates on that one source file.
- **Directory**: recursively finds every file matching a registered
  analyzer extension (currently `.swift`) under that directory and runs
  the command on each one. Used for whole-project batch splitting.

## Flags

| Flag | Applies to | Effect |
|---|---|---|
| `--flat` | `preview`, `split` | Disable kind subfolders. Puts all split files directly in `<Type>/` instead of `<Type>/Functions/`, `<Type>/Properties/`, `<Type>/Types/`. Default is many-folders (on), matching the `/god`/`/macdbg` sub-cluster pattern. |
| `--yes` | `split` | Skip the confirmation prompt. Required for non-interactive/scripted runs. |
| `--ai` | `analyze`, `preview`, `split` | **Stub.** No AI provider is wired up yet — prints a notice and falls back to the heuristic parser. Intended future hook for smarter member naming/grouping once a provider + API key are configured. |
| `--extra NAME[,…]` | `extract` | Force named locals/params into every helper even when the block source text never writes them (escape hatch for names only visible after macro expansion). Prefer the built-in `MACRO_IMPLIES` table for known TimGui_DEBUG_LOG_* cases. |
| `--depth N` / `--depth Lo-Hi` | `blocks` | Analyse cuts at brace depth up to N (default 1), or a depth range. |
| `--search DIR` | `audit-inline` | Extra include-search root (repeatable), matching the compiler's `-I` path. |
| `--only-under SUBSTR` | `audit-inline` | Only recurse into includes whose resolved path contains SUBSTR (e.g. `TimGui`). |

## Examples

```bash
# single file, read-only
qs analyze  MyView.swift
qs preview  MyView.swift

# single file, write (interactive confirm)
qs split    MyView.swift

# single file, write, no prompt
qs split    MyView.swift --yes

# flat structure instead of Functions/Properties/Types subfolders
qs split    MyView.swift --flat --yes

# whole project, read-only batch preview
qs preview  src/

# whole project, write, no prompt (destructive — backup first!)
qs split    src/ --yes

# past the mechanical floor: analyse then lift sub-helpers from ONE function
qs blocks   TimGui_widgets/TabItemEx.h
qs extract  TimGui_widgets/TabItemEx.h 180:220:HandleTabDragAndDrop --extra g
qs audit-inline TimGui_widgets/TabItemEx.h ~/.Trash/TabItemEx.h.mb-162200 \
    --search src/TimGui --only-under TimGui
```

## Safety notes

- `split` always backs up the original file to `~/.Trash/<name>.qs-backup-<timestamp>` before/after writing — nothing is deleted.
- Directory mode in `split` shows every file's full proposed plan before asking ONE combined confirmation (or `--yes` to skip it).
- Take a full project backup (`cp -R`) before running `split` on a directory — the tool's parser is heuristic, not a full compiler front end. Always read `preview` output first.
