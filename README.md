# ainative-rank — public leaderboard

Two lists. Both populated by **opening issues** in this repo — no forks, no PRs, no branches.

→ Take the test: https://rank.ha7ch.com

## `participants.md`

Everyone who's taken the test, regardless of grade.

During the test, your agent opens an issue:

```
gh issue create --repo HA7CH/ainative-rank-leaderboard --title "+@your-handle"
```

A bot watches for issues whose title starts with `+@` and appends a row to `participants.md`:

```
| [@your-handle](https://github.com/your-handle) | 2026-05-17 |
```

## `leaderboard.md`

The ranked list, by grade. **Opt-in** — only candidates who choose to publish their grade appear.

After the test finishes, the CLI shows the issue command. Run it if you want your grade public:

```
gh issue create --repo HA7CH/ainative-rank-leaderboard --title "+leaderboard @your-handle S 92.4"
```

Bot watches for `+leaderboard` issues and appends:

```
| [@your-handle](https://github.com/your-handle) | S | 92.4 |
```

Skip the second issue if you'd rather keep the grade private — the participants entry stays either way.
