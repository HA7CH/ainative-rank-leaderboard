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

This works cleanly when you drive the test through **Claude Code** (CLI **or** Desktop — both write the same JSONL into `~/.claude/projects/<encoded-cwd>/`) in the **same directory** where you ran `start`. It fails in a handful of common cases below.

> **Aside — what about Desktop's other directories?** (macOS-verified) Claude Code Desktop also writes to `~/Library/Application Support/Claude/claude-code-sessions/`, `claude-code-vm/`, and `local-agent-mode-sessions/`. Those hold Desktop session metadata, the Code-mode VM state, and agent-mode / sub-agent payloads — **not** the transcript the grader reads. Don't point `CLAUDE_TRANSCRIPT_FILE` at any of them; they aren't Claude JSONL. The transcript the grader wants is still the `*.jsonl` under `~/.claude/projects/`, which Desktop is already writing for you.

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

### Case 3 — Your main AI tool isn't Claude Code at all

The grader expects Claude Code JSONL. **Claude Code CLI and Claude Code Desktop both produce that** (they share `~/.claude/projects/<encoded-cwd>/*.jsonl`), so neither needs special handling. The real "wrong tool" cases are **Codex** (CLI or Desktop), **Cursor**, and other non-Claude agents — those write to completely different stores in completely different formats, and `locateTranscript()` returns null even when you have plenty of AI-native activity on disk.

For reference, here's where those tools keep local logs:

> ⚠️ **Platform note — all the paths below were verified on macOS.** The home-directory stores (`~/.claude/`, `~/.codex/`) should look the same on Linux. On Windows, expect the same *shape* under `%USERPROFILE%\` (e.g. `%USERPROFILE%\.claude\projects\`, `%USERPROFILE%\.codex\sessions\`), but I haven't tested them and Case 2 already documents one Windows-only encoding bug — assume there may be others. The `~/Library/Application Support/...` rows are **macOS-only paths**; the equivalents on Windows are under `%APPDATA%\` and on Linux under `~/.config/` or `~/.local/share/`, but the directory names may differ between platforms and I haven't verified them.

| Tool                            | Local store (macOS-verified)                                                             | Format       |
| ------------------------------- | ---------------------------------------------------------------------------------------- | ------------ |
| Claude Code (CLI **+** Desktop) | `~/.claude/projects/<encoded-cwd>/*.jsonl`                                               | Claude JSONL |
| Codex (CLI **+** Desktop)       | `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` (active) + `~/.codex/archived_sessions/rollout-*.jsonl` (history), indexed by `~/.codex/state_5.sqlite` (`threads` table → `rollout_path`) | Codex JSONL + SQLite |
| Cursor                          | `~/Library/Application Support/Cursor/User/workspaceStorage/<hash>/`                     | LevelDB / SQLite |

> **Two paths often listed as "Codex transcripts" that aren't** (macOS-verified):
> - `~/.codex/logs_2.sqlite` is the Codex **application log** (`logs(ts, level, target, module_path, file, line, …)`), not conversation content. It can grow into the multi-GB range and looks juicy, but it doesn't contain the rollouts.
> - `~/Library/Application Support/Codex/` only holds the Electron app's standard caches (Cookies, Local Storage, GPUCache, …) plus a sidebar config. Codex Desktop writes its sessions to `~/.codex/`, the same place the CLI uses.
>
> **Not in this table on purpose** (macOS-verified): `~/Library/Application Support/Claude/IndexedDB/https_claude.ai_0.indexeddb.leveldb/` belongs to the `claude.ai` web chat surface (it's the front-end cache for the chat UI), not to Claude **Code**. It has nothing to do with grading and shouldn't be treated as a transcript source.

None of the non-Claude stores are drop-in replacements for `CLAUDE_TRANSCRIPT_FILE` — the grader parses Claude's JSONL shape (`uuid`, `type`, `timestamp`, …). Until non-Claude sources are supported natively (tracked in [#45](https://github.com/HA7CH/ainative-rank-leaderboard/issues/45)), the practical options are:

1. **Run the four stages through Claude Code** (CLI or Desktop — either works, they share the same JSONL store) so the test itself produces a real Claude JSONL. Your non-Claude sources can still inform what your agent says during stage 1, but the file the grader reads is the one Claude Code writes.
2. **Convert and serve** — point `CLAUDE_TRANSCRIPT_FILE` at a JSONL you've translated from your non-Claude source into the Claude shape. Anything beyond a one-off script is out of scope here.

## Quick sanity check before `finish`

```bash
node -e 'const f=process.env.CLAUDE_TRANSCRIPT_FILE; if(!f){console.log("CLAUDE_TRANSCRIPT_FILE not set, CLI will fall back to ~/.claude/projects/<cwd>")} else {const fs=require("fs"); console.log(f, fs.existsSync(f)?"OK":"MISSING")}'
```

If that prints `OK` (or your fallback directory has a recent `*.jsonl`), `finish` should ship a real transcript.

## Related issues

- [#37](https://github.com/HA7CH/ainative-rank-leaderboard/issues/37) — Windows: transcript auto-detect always misses
- [#45](https://github.com/HA7CH/ainative-rank-leaderboard/issues/45) — Cursor + Codex workflows graded as empty transcripts
