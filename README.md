# reviewer-eval-skill

A Claude Code skill named `reviewer-eval` for evaluating AI code reviewers using bugs your team already fixed.

Rewind the code to before a known fix, hide the fix, let the reviewer review it like a real pull request, and grade whether it named the real bug. Clean changes count false alarms.

## What it does

Teaches your coding agent to:

- Build a regression suite from bugs your team already fixed.
- Compare reviewers and review bots on bugs caught, false alarms, speed, and cost.
- Tune a reviewer with a held-out split.
- Decide whether a paid review bot is worth keeping.

## Install

```sh
git clone https://github.com/manty/reviewer-eval-skill ~/.claude/skills/reviewer-eval
```

Claude Code picks it up on the next session.

To update:

```sh
git -C ~/.claude/skills/reviewer-eval pull
```

## Example prompts

- "which AI reviewer should we use for our repos"
- "is our review bot worth keeping"
- "a new model came out, should we switch our PR reviewer"
- "hillclimb our review prompt without overfitting"

## Scope and data

This repo contains no code or data and has no vendor affiliation. Your agent builds the runner in your own repo.

Running it sends your code to the reviewers you choose, so check your data rules first.

## Origin

Written after a three-day test of 7 AI reviewers plus a commercial review bot on a real codebase. Includes the traps that test hit.

## License

MIT.