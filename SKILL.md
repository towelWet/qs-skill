---
name: qs
description: Quick Split — CLI tool that analyzes a source file and splits it into a folder-per-type structure with one file per function/computed-property/nested-type, following /split-files + /dynamic-code + /hot-loadable discipline. Language support is hot-loadable via analyzers/*.py — Swift ships first, more languages drop in as new analyzer files with zero core edits. Use when the user says /qs, "quick split", "split this file", or wants an automated first pass before manual /modularize refinement.
---

# qs — Quick Split

Automated monolith splitter. Point it at one file, it proposes (or writes) a
folder-per-type structure matching the `/god` and `/macdbg` reference
granularity: one function/computed-var/nested-type per file, shell file keeps
only the type declaration + stored properties + init.

## Usage

```
~/.claude/skills/qs/bin/qs analyze <file>                # list split candidates, no writes
~/.claude/skills/qs/bin/qs preview <file>                # show proposed folder + file contents, no writes
~/.claude/skills/qs/bin/qs split <file> [--verify]       # write the split; --verify runs swiftc -parse + round-trip
~/.claude/skills/qs/bin/qs stats <dir>                   # line-count stats + oversized file list
~/.claude/skills/qs/bin/qs batch <dir> --max-lines N    # split every oversized file
~/.claude/skills/qs/bin/qs batch-undo <txid>            # undo a whole batch by transaction id
~/.claude/skills/qs/bin/qs undo <file>                  # restore the last backup from ~/.Trash
~/.claude/skills/qs/bin/qs blocks <file> [--depth N]     # SAFE sub-helper candidates past mechanical floor
~/.claude/skills/qs/bin/qs extract <file> A:B:Name ...   # lift those ranges into static helpers (verbatim body)
~/.claude/skills/qs/bin/qs audit-inline <cur> <backup>   # multiset reassembly audit (catches overwrite loss)
~/.claude/skills/qs/bin/qs routes <file>                 # extract /api/ routes from JS/TS (minified-safe)
```

Common flags: `--verify`, `--min-lines N`, `--allow-dirty`, `--flat`, `--extra`, `--depth`.

Add `~/.claude/skills/qs/bin` to `PATH` to just run `qs analyze <file>`.

## Architecture (hot-loadable, dynamic)

```
qs/
  bin/qs                    entry point, resolves its own dir at runtime
  core/
    cli.py                  arg parsing, dispatches to discovered command
    discover_commands.py    globs commands/*.py, no hardcoded command list
    discover_analyzers.py   globs analyzers/*.py, no hardcoded language list
    file_writer.py          writes plan, backs up original via mv to ~/.Trash
    walk.py                 resolves a file/dir arg into matching source files
    swift_parser.py         heuristic brace-depth parser for Swift
  commands/
    analyze.py               COMMAND manifest: {"name": "analyze", "run": fn}
    preview.py
    split.py
    help.py
  analyzers/
    swift.py                 ANALYZER manifest: {"id","extensions","analyze","propose"}
    python.py                Python analyzer (top-level def/class)
    javascript.py            JS/TS analyzer (.js/.mjs/.cjs/.ts/.jsx/.tsx)
    # drop rust.py / go.py here later — zero core edits
```

**Adding a new language:** drop `analyzers/<lang>.py` exporting an `ANALYZER`
dict with `id`, `extensions`, `analyze(path)`, `propose(path)`. No edits to
any existing file. Same for new commands under `commands/`.

## Swift analyzer (v2)

Heuristic brace-depth parser (`core/swift_parser.py`), not a full compiler
front end:
- Finds **all** top-level `class`/`struct`/`enum` declarations in the file.
- Picks the **primary type** by matching the filename stem (e.g.
  `SettingsView.swift` → `SettingsView`), not just the first declaration.
- Splits top-level `func`, computed `var` (body opens with `{`, not `=`),
  and nested `class`/`struct`/`enum` into their own files.
- `func`/`var` members become `extension <Type> { ... }` in their own file.
- Nested types are wrapped in `extension <Type> { ... }` so they remain
  `Type.Nested` (e.g. `PlanExecutor.TemporaryAccessError`).
- Sibling top-level types (small subview structs declared in the same file)
  are extracted as standalone files too.
- File names are prefixed with the parent type name so every generated file
  is unique within the module (e.g. `PlanExecutorExecute.swift`,
  `ConfigStoreSave.swift`). This is required because Swift uses basenames,
  not paths, to distinguish private declarations and Xcode driver artifacts.
- `private`/`fileprivate` (without `(set)`) are widened to `internal` so
  members remain accessible after moving across files.
- `private(set)`/`fileprivate(set)` are widened to `internal(set)` so setters
  remain reachable from mutators that may move to other files.
- `override func`, `required init`, and other extension-forbidden members
  are kept in the shell with a warning.
- Leading doc comments and attributes (`@MainActor`, `@available`, etc.) are
  preserved with the member when it is extracted.
- `body`, stored properties, `init`, and unmatched code stay in the shell
  file (named after the original file, moved into `<Type>/`).
- Default output groups files into `<Type>/Functions/`, `<Type>/Properties/`,
  `<Type>/Types/`; `--flat` disables the subfolders.

**Known limitations:** single-line string/comment stripping only (no
multi-line string literals), computed vars must open `{` on the same line
as the declaration. Always run `preview` and read the output before `split`.

## JavaScript / TypeScript analyzer

Handles `.js/.mjs/.cjs/.ts/.jsx/.tsx`. Splits **top-level** `function`,
`class`, and `const/let/var NAME = …{ block }` declarations into
`Functions/` and `Types/`, leaving a barrel `index.<ext>` that re-exports
them so existing importers keep working.

- Parsing is brace-depth over a **string/comment-masked** copy of the source,
  so `{`/`}` and keywords hiding inside `"strings"` or `/* comments */` never
  skew the count.
- A blockless binding (`const X = "…";`) is correctly skipped — it will not
  swallow the next statement's block.
- Known limitation: regex literals `/.../ ` are not tracked (noted in output).
- **Minified/webpack bundles do not split**: everything lives inside one IIFE
  with mangled names, so there are no top-level declarations. The analyzer
  detects bundles via `_detect_bundle` heuristics (large byte size + few lines,
  max line length > 500, or `__webpack_modules__` / `webpackBootstrap` markers)
  and returns an actionable error instead of a vague "no top-level" message.
  Rule `never-qs-minified-bundles`:
  treat compiled output as API reference only — port from readable source or
  build fresh in split-files architecture.
- **`qs routes <file>`** — regex extraction of `app.get/post/…("/api/…")` and
  bare `/api/…` string literals. Works on minified bundles; use it to list the
  API surface before porting. Does not write files.

## C / C++ analyzers

`cpp.py` (CppIdiom) lifts every member/free function into its own fragment header
`#include`d back at its original position, so compiled text is unchanged.
`c.py` splits plain C. Hardened on JUCE → TUCE (2026-10-09):

| Rule | Why |
|---|---|
| one member per fragment | user rule: one function per file |
| name = param-list owner; `X_ctor`/`X_dtor` | old rule read ctor initializer lists |
| overloads named by param types | no `_2` position suffixes |
| case-insensitive dedup | APFS/exFAT treat `getX.h` = `GetX.h` |
| `#define` bodies opaque | lifting cut JUCE macros in half |
| blocks with `#include` stay | relative includes break one folder deeper |
| `Foo.cpp` → `Foo_cpp/`, `Foo.h` → `Foo/` | .h/.cpp twins overwrote each other |
| `class A::B` → folder `B` | nested out-of-line classes collided |
| `batch --exclude GLOB` | keep vendored third-party untouched |
| walk skips `._*` | exFAT AppleDouble sidecars are binary |

Multi-class files need repeated `batch` passes (one container per pass).
Always prove with a compile + `audit-inline` against the pristine file.

## Past the mechanical floor (`blocks` / `extract`)

When every multi-member file is already one-function-per-file, further shrinkage
needs **sub-helper extraction inside a single function body** — a different tool
from `split`. That lives in qs (not a scratchpad):

- `qs blocks <file>` — blank-line cuts at brace depth 1; tags `<< SAFE` when the
  block has no return/goto/break/continue and declares nothing later code reads.
- `qs extract <file> start:end:Name ...` — lifts those ranges into
  `stem/Name.h` static helpers; body text moves **verbatim**; params are always
  by reference (arrays decay to pointers). Backs up to `~/.Trash/<name>.mb-*`.
- Built-in `MACRO_IMPLIES` table: `TimGui_DEBUG_LOG_TABLE` (and friends) expand
  to code that references `g` — without this, the identifier scan of source text
  silently omits `g`. `--extra NAME` is the escape hatch for unknown macros.
- Multi-declarator locals (`bool hovered, held, pressed;`) are all visible —
  DECL accepts a trailing comma so the second/third names are not dropped, and
  the declarator scan blanks paren/bracket interiors first so call-argument
  commas (`bool pressed = ButtonBehavior(bb, id, ...)`) are never parsed as
  extra declarators.
- Visibility is **scope-aware** (declarations die when their brace depth
  closes; latest live declaration wins), knows **for-loop init/range variables**
  (`for (int order_n = 0; ...)` — activated when the loop body is entered), and
  rejects keyword pseudo-declarations (`return is_open;` is not a decl of
  `is_open`). Local names may start uppercase (`float H, S, V`).
- `blocks` escape detection scans the block's **own** lines for declarations —
  a same-named shadow in an earlier sibling scope must not make a block look
  like it declares nothing. A depth-floor guard also rejects ranges that dip
  below their start depth (brace-balanced `} else {` spans are unliftable).
- Extraction hosts can themselves be extract-generated (`namespace X {`
  wrappers add a base depth; `Type*&` params parse), so extraction **recurses**.
- A later pass's `#include`s are appended AFTER existing ones — a new helper may
  call a sibling lifted earlier, and static-function order in the TU matters.
- `qs audit-inline <current> <backup>` — inlines fragment `#include`s on **both**
  sides and diffs meaningful-line **multisets** (catches overwritten fragments
  that set-membership round-trips miss).

## Companion skills

`/split-files` (the rule), `/modularize` (broader multi-language automated
splitter), `/dynamic-code`, `/hot-loadable`, `/framework-swift`, `/cli-first`.
`qs` is the Swift-first, minimal, single-file-at-a-time implementation of
that stack — run it, then hand-review with `/split-files` discipline.

## Xcode build-system notes (learned from Towel Mac Organizer)

After splitting, classic Xcode projects with explicit `PBXFileReference` lists
need two things to compile cleanly:

1. **Convert the `src` group to a PBXFileSystemSynchronizedRootGroup** — Xcode
   16+ discovers files automatically and uses full relative paths for derived
   build artifacts, avoiding basename collisions. A helper script lives in
   `scripts/convert-src-to-synchronized-group.rb` in the project.
2. **Set `SWIFT_EMIT_LOC_STRINGS = NO`** — Xcode's String Catalog extraction
   derives intermediate files by basename, so two `HeaderView.swift` files in
   different folders cause a "Multiple commands produce" failure even if the
   filenames are otherwise valid. The project build script needs this disabled.

These two fixes together make the project fully hot-loadable: drop any new
file under `src/` and it builds automatically with zero pbxproj edits.
