# qs-skill

**Claude Code skill for [qs (Quick Split)](https://github.com/towelWet/qs) — automated monolith splitter.**

This is the AI skill that drives Claude Code when splitting oversized source files.
Holds command reference, analyzer docs per language, architecture notes,
and known limitations so context never has to be re-derived.

## What this skill contains

| File | Purpose |
|---|---|
| `SKILL.md` | full usage guide — commands, flags, analyzer docs, Xcode notes |
| `FLAGS.md` | flag reference for all commands |

## Relationship to qs project

This skill is the **usage guide and AI integration layer**.
Source code (analyzers, commands, core) lives in the project repo:

**Project repo:** [towelWet/qs](https://github.com/towelWet/qs)

| Concern | Where |
|---|---|
| Usage guide, flag reference, known limits | this repo (`qs-skill`) |
| Python source, analyzers, commands | [towelWet/qs](https://github.com/towelWet/qs) |

## Install

The skill needs qs code beside it (`bin/qs` is what SKILL.md calls):

```
git clone https://github.com/towelWet/qs ~/.claude/skills/qs
git clone https://github.com/towelWet/qs-skill /tmp/qs-skill
cp /tmp/qs-skill/SKILL.md /tmp/qs-skill/FLAGS.md ~/.claude/skills/qs/
```

Then invoke `/qs` in Claude Code.

## License

MIT.
