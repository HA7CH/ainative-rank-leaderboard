# Finding your transcript

If `npx @ha7ch/ainative-rank finish` prints:

```
Couldn't find a Claude Code transcript — submitting an empty session.
```

…you'll get **D 0.0** regardless of how the four stages went. This doc explains why, and how to point the CLI at the right file before re-running `finish`.

## What the CLI is looking for

By default `finish` runs `locateTranscript()`, which in turn checks:

1. The `CLAUDE_TRANSCRIPT_FILE` environment variable (if set and the file exists, use it).
2. Otherwise: `~/.claude/projects/<encoded-cwd>/*.jsonl`, where `<encoded-cwd>` is the current working directory with `/` replaced by `-`.

It then reads the **most recently modified** `.jsonl` in that directory.

This works cleanly when you've been driving the whole test through **Claude Code CLI**, in the **same directory** where you ran `start`. It fails in a handful of common cases below.

## When the auto-locate misses

### Case 1 — You ran the test in one directory, but did your real work in another

Example: you ran `start` in `~/Desktop/rank/`, but your last week of Claude Code work lives in `~/code/myproject/`.

`finish` only looks under the encoded form of `~/Desktop/rank/`, so it sees the four-stage transcript and nothing else (or nothing at all, if the test directory is brand new).

**Fix.** Point `CLAUDE_TRANSCRIPT_FILE` at the JSONL from your real working directory before running `finish`:

```bash
# macOS / Linux — find the newest jsonl across all Claude Code projects
export CLAUDE_TRANSCRIPT_FILE="$(ls -t ~/.claude/projects/*/*.jsonl | head -1)"
npx @ha7ch/ainative-rank finish
```

Or pick a specific project directory:

```bash
# Replace /Users/you/code/myproject with the real cwd of your work
encoded=$(echo "/Users/you/code/myproject" | sed 's|/|-|g')
export CLAUDE_TRANSCRIPT_FILE="$(ls -t ~/.claude/projects/$encoded/*.jsonl | head -1)"
npx @ha7ch/ainative-rank finish
```

### Case 2 — You're on Windows

There's a known cwd-encoding bug on Windows (see [#37](https://github.com/HA7CH/ainative-rank-leaderboard/issues/37)): `process.cwd()` returns `D:\AI\ask`, the encoder only replaces `/`, so the CLI looks for `~/.claude/projects/D:\AI\ask/` — a path that can't exist on NTFS. The real directory Claude Code creates is `D--AI-ask`.

**Workaround.** Point `CLAUDE_TRANSCRIPT_FILE` at the actual file:

```powershell
# PowerShell — list candidates, then pick the latest jsonl
Get-ChildItem "$HOME\.claude\projects\*\*.jsonl" |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1 -ExpandProperty FullName
# Then:
$env:CLAUDE_TRANSCRIPT_FILE = "<path you just printed>"
npx @ha7ch/ainative-rank finish
```

### Case 3 — Your main AI tool isn't Claude Code CLI

The grader expects Claude Code JSONL. If your real workflow lives in **Claude Desktop**, **Codex** (CLI or Desktop), **Cursor**, or another agent, `locateTranscript()` returns null even though you have plenty of AI-native activity on disk.

For reference, here's where those tools keep local logs on macOS:

| Tool                | Local store                                                                              | Format       |
| ------------------- | ---------------------------------------------------------------------------------------- | ------------ |
| Claude Code CLI     | `~/.claude/projects/<encoded-cwd>/*.jsonl`                                               | Claude JSONL |
| Claude Desktop      | `~/Library/Application Support/Claude/IndexedDB/https_claude.ai_0.indexeddb.leveldb/`    | LevelDB      |
| Codex CLI / Desktop | `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`, plus `~/.codex/logs_2.sqlite`            | Codex JSONL / SQLite |
| Cursor              | `~/Library/Application Support/Cursor/User/workspaceStorage/<hash>/`                     | LevelDB / SQLite |

None of those are drop-in replacements for `CLAUDE_TRANSCRIPT_FILE` — the grader parses Claude's JSONL shape (`uuid`, `type`, `timestamp`, …). Until non-Claude sources are supported natively (tracked in [#45](https://github.com/HA7CH/ainative-rank-leaderboard/issues/45)), the practical options are:

1. **Run the four stages through Claude Code CLI** so the test itself produces a real Claude JSONL. The other sources still inform what your agent says during stage 1, but the file the grader reads is the one Claude Code writes.
2. **Convert and serve** — point `CLAUDE_TRANSCRIPT_FILE` at a JSONL you've translated from your non-Claude source into the Claude shape. Anything beyond a one-off script is out of scope here.

## Quick sanity check before `finish`

```bash
node -e 'const f=process.env.CLAUDE_TRANSCRIPT_FILE; if(!f){console.log("CLAUDE_TRANSCRIPT_FILE not set, CLI will fall back to ~/.claude/projects/<cwd>")} else {const fs=require("fs"); console.log(f, fs.existsSync(f)?"OK":"MISSING")}'
```

If that prints `OK` (or your fallback directory has a recent `*.jsonl`), `finish` should ship a real transcript.

## Related issues

- [#37](https://github.com/HA7CH/ainative-rank-leaderboard/issues/37) — Windows: transcript auto-detect always misses
- [#45](https://github.com/HA7CH/ainative-rank-leaderboard/issues/45) — Cursor + Codex workflows graded as empty transcripts
