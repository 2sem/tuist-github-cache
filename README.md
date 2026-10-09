# tuist-github-cache

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code) for speeding up **Tuist** build, test and deploy jobs on **GitHub Actions**.

## What it covers

1. **Dependency cache** — `actions/cache` for `Tuist/.build` and `~/.cache/swifterpm`, keyed on `Tuist/Package.resolved`, including the stale-key trap when the cached paths change.
2. **Tuist binary cache** — OIDC `tuist auth login`, `tuist setup cache`, `enableCaching`.
3. **Skip/cancel wasted runs** — `paths:` filters and `concurrency:`.
4. **Action upgrades** — Node 24 versions of checkout, cache and mise-action.
5. **Verification** — how to confirm hit/miss and measure the gain.

## Install

```bash
git clone https://github.com/2sem/tuist-github-cache.git ~/.agents/skills/tuist-github-cache
ln -s ../../.agents/skills/tuist-github-cache ~/.claude/skills/tuist-github-cache
```

## Use

Ask Claude Code: "speed up Tuist CI with caching", "add swifterpm cache to the workflows", or "my tuist cache never hits".

See [`SKILL.md`](./SKILL.md) for the full steps.
