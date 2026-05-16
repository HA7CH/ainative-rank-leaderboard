# ainative-rank — public leaderboard

Two lists, two different things.

→ Take the test: https://rank.ha7ch.com

## `participants.md`

Anyone who's taken the test, regardless of grade.

You add yourself here during the test (your agent will be prompted to open the PR). One row, no grade. This is the proof-of-life list — it shows up before grading finishes, so we don't know yet how you did.

Row format:

```
| [@your-handle](https://github.com/your-handle) | 2026-w20 | 2026-05-17 |
```

## `leaderboard.md`

The ranked list, by grade.

Adding yourself here is **opt-in** — after the test finishes and your grade is computed, the CLI will ask if you want to publish it. If yes, your agent opens a second PR. Only people who chose to publish appear.

Row format:

```
| [@your-handle](https://github.com/your-handle) | 2026-w20 | S | 92.4 |
```

Skip it if you don't want your grade public. Nothing about your private session changes either way.
