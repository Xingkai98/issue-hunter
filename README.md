# Issue Hunter

Daily AI-powered open source issue discovery, deep analysis, and implementation. Scans 10 priority repos + user-curated watchlist + trending repos, finds 15 high-quality issues, analyzes them, implements the top 5 by merge probability, and produces a handoff doc for review.

## How it works

Every day at 4:00 AM, an AI agent:

1. **Multi-source discovery** — scans 10 priority repos (`repos.json`), user-curated watchlist (`watchlist.json`), GitHub trending, and global search
2. **Score & filter** — evaluates 25-dimension scoring across technical depth, maintainer engagement, community interest, scope clarity, merge probability
3. **Community assessment** — reads full comment threads; checks for existing implementations, timeliness, maintainer stance, CLA barriers
4. **Select 15** — 5 from priority repos (Tier A) + 10 from watchlist + exploration (Tier B)
5. **Deep analysis** — background, root cause, fix strategy for all 15
6. **Implement top 5** — fork, branch, code, self-verify (lint, type-check, tests), push to personal fork. No PR without approval.
7. **Handoff doc** — generates `/tmp/issue-hunter-handoff-YYYY-MM-DD.md` for review
8. **Report** — writes `reports/YYYY-MM-DD-HHmm.md` and sends briefing via Feishu

## Configuration

- `repos.json` — 10 priority repos for Tier A (editable, tracked)
- `watchlist.json` — user-curated watchlist for Tier B (editable, tracked)
- `config.json` — Feishu IDs (gitignored, template at `config.example.json`)

## Output

Each report covers 15 issues with:
- Project background (what it does, stars, language)
- Problem description (what's broken, user impact, repro steps)
- Fix strategy design (root cause, files to modify, risk assessment, alternatives)

## Repo structure

```
~/issue-hunter-loop/
├── repos.json         # Configurable priority repo list
├── config.example.json # Feishu config template
├── CLAUDE.md          # Full agent workflow instructions
├── state.json         # Structured dedup — all processed issues
├── COMPLETED.md       # Human-readable log
├── reports/           # Daily analysis reports
└── repos/             # Cloned repos (cleanup periodically)
```

## Why research-only

Many "good first issue" tools push to fix-and-PR immediately. That leads to:
- Low-quality PRs on repos you don't understand
- Duplicate work (someone already has a fix)
- Wasted time on stale or wont-fix issues

Issue Hunter prioritizes understanding first. The analysis tells you whether an issue is worth your time before you write a single line of code.

## Stack

- GitHub API (via `gh` CLI)
- GitSense for automated issue discovery
- Feishu/Lark for briefing delivery
- Claude Code for AI analysis
