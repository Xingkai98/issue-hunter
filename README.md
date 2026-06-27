# Issue Hunter

Daily AI-powered open source issue discovery and analysis. Scans trending AI/Agent/LLM repos, finds high-quality issues, and produces structured analysis reports — no code, just insight.

## How it works

Every day at 4:00 AM, an AI agent:

1. **Multi-source discovery** — scans priority repos (openclaw, pi, hermes-agent, nanochat, minimind, nanobot), GitHub trending, GitSense, and global issue search
2. **Score & filter** — evaluates issues on technical depth, maintainer engagement, community interest, scope clarity, and analysis value
3. **Community assessment** — reads full comment threads to check for existing implementations, timeliness, and maintainer stance
4. **Deep analysis** — for the top 3 issues: background, root cause, fix strategy with specific code paths
5. **Report** — writes `reports/YYYY-MM-DD-HHmm.md` and sends a briefing via Feishu

## Output

Each report covers 3 issues with:
- Project background (what it does, stars, language)
- Problem description (what's broken, user impact, repro steps)
- Fix strategy design (root cause, files to modify, risk assessment, alternatives)

## Repo structure

```
~/issue-hunter-loop/
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
