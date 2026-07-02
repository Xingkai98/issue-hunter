# Issue Hunter Loop — Research & Implementation

Research + implement: discover trending AI/Agent/LLM projects, find 15 high-quality issues (5 Tier A + 10 Tier B), analyze, then implement the top 5 with highest merge probability. Produce a report + handoff doc.

## Configuration

- `config.json` — Feishu user open_id + drive folder token (gitignored, see `config.example.json`)
- `repos.json` — Tier A repo list, editable (tracked, see `repos.example.json` for format)
- `watchlist.json` — user-curated watchlist for Tier B, editable (tracked, see `watchlist.example.json`)

## Core Rules

- **所有产出（报告、简报、分析）必须使用简体中文**
- Work inside `~/issue-hunter-loop/`
- Track all processed issues in `state.json` — NEVER re-process
- One report per loop iteration, **always 15 issues** (5 Tier A + 10 Tier B)
- From the 15, select top 5 by merge probability and **actually implement them**
- Output: report → `reports/YYYY-MM-DD-HHmm.md`, handoff → `/tmp/issue-hunter-handoff-YYYY-MM-DD.md`

---

## Phase 1: Discovery

### 1a. Tier A — repos.json (10 repos)

```bash
REPOS=$(python3 -c "import json; print(' '.join(r['repo'] for r in json.load(open('$HOME/issue-hunter-loop/repos.json'))))" 2>/dev/null)
for repo in $REPOS; do
  gh api "repos/$repo/issues?labels=bug&state=open&per_page=8&sort=updated" \
    --jq '.[] | "#\(.number) [👍\(.reactions.total_count)] [💬\(.comments)] [\(.updated_at[:10])] \(.title)"' &
  gh api "repos/$repo/issues?labels=enhancement&state=open&per_page=3&sort=updated" \
    --jq '.[] | "#\(.number) [👍\(.reactions.total_count)] [💬\(.comments)] [\(.updated_at[:10])] \(.title)"' &
done
wait
```

### 1b. Tier B source 1 — watchlist.json

```bash
WATCHLIST=$(python3 -c "import json; print(' '.join(r['repo'] for r in json.load(open('$HOME/issue-hunter-loop/watchlist.json'))))" 2>/dev/null || echo "")
if [ -n "$WATCHLIST" ]; then
  for repo in $WATCHLIST; do
    gh api "repos/$repo/issues?labels=bug&state=open&per_page=6&sort=updated" \
      --jq '.[] | "#\(.number) [👍\(.reactions.total_count)] [💬\(.comments)] [\(.updated_at[:10])] \(.title)"' &
    gh api "repos/$repo/issues?labels=enhancement&state=open&per_page=3&sort=updated" \
      --jq '.[] | "#\(.number) [👍\(.reactions.total_count)] [💬\(.comments)] [\(.updated_at[:10])] \(.title)"' &
  done
  wait
fi
```

### 1c. Tier B source 2 — GitHub Trending + global search

```bash
# Trending AI repos
gh api "search/repositories?q=stars:>3000+pushed:>$(date -d '30 days ago' +%Y-%m-%d)+topic:ai&sort=stars&order=desc&per_page=10" \
  --jq '.items[] | "\(.full_name) [⭐\(.stargazers_count)] [push:\(.pushed_at[:10])] \(.description[:120])"'

# Fast-growing 2026 agent repos
gh api "search/repositories?q=created:>2026-01-01+stars:>2000+topic:agent&sort=stars&order=desc&per_page=8" \
  --jq '.items[] | "\(.full_name) [⭐\(.stargazers_count)] [创建:\(.created_at[:10])] \(.description[:120])"'

# Global bug search (spam-filtered)
gh search issues --label bug --state open --sort updated --limit 25 \
  --json number,title,url,repository,commentsCount,updatedAt | python3 -c "
import json, sys
data = json.load(sys.stdin)
skip = {...}  # maintained spam list
for d in data:
    r = d['repository']['nameWithOwner']
    if any(s.lower() in r.lower() for s in skip): continue
    print(f\"{r}#{d['number']} [{d['commentsCount']}💬] {d['title'][:130]}\")
"
```

**Dynamic expansion**: discover new fast-growing AI/Agent/LLM repos (stars > 10k, growth > 1k/week). Add to `repos.json` (if core) or `watchlist.json` (if niche/experimental).

---

## Phase 2: Filter, Score, Community Assessment

### Must-pass filters
- Stars > 100 (or well-known org)
- Pushed within last 30 days
- Language: Go, Python, Rust, TypeScript
- NOT a hacktoberfest/spam/template repo
- NOT already in `state.json`

### Scoring (1-5 each, total /25)
| Dimension | 5 points | 1 point |
|-----------|----------|---------|
| Technical depth | Real bug with repro steps, stack traces | Vague description |
| Maintainer engagement | OWNER/MEMBER commented | Zero engagement |
| Community interest | 3+ 👍, multiple confirmations | Zero reactions |
| Scope clarity | Clear acceptance criteria, file hints | No boundaries |
| Merge probability | Simple fix, active maintainer, no CLA barrier | Complex, CLA required, inactive |

### Community Status Assessment (MANDATORY)
- Read full comment thread for each shortlisted candidate
- **Existing implementation?** → if quality PR exists, demote/skip
- **Timeliness:** last activity < 7 days (+1), > 60 days (-2)
- **Maintainer stance:** "lgtm"/"PR welcome" → strong signal; "wont fix" → skip
- **CLA/contributor gate?** → note in assessment, affects merge probability

---

## Phase 3: Select 15 Issues

**Tier A — 5 from repos.json:** highest-scoring issues across diverse repos (max 2 from same repo)

**Tier B — 10 total:**
- 5 from watchlist.json (or exploration if watchlist is empty/insufficient)
- 5 from any other repos (GitHub trending, global search, serendipitous discovery)

**Diversity rules:** no more than 3 from same repo across all 15. Mix of bugs and enhancements. At least 3 different languages.

---

## Phase 4: Deep Analysis

For each of the 15 issues:
- **背景：** What is this project? What subsystem is affected?
- **问题：** What exactly is broken? User impact? Repro steps?
- **修复方案：** Root cause, files to modify, risk assessment, alternative approaches
- **合入评估：** Merge probability reasoning (maintainer stance, CLA, scope)

---

## Phase 5: Write Report

Write to `~/issue-hunter-loop/reports/YYYY-MM-DD-HHmm.md`. Chinese output.

---

## Phase 6: Upload to Feishu Drive

```bash
FOLDER_TOKEN=$(python3 -c "import json; print(json.load(open('$HOME/issue-hunter-loop/config.json'))['feishu']['drive_folder_token'])")
lark-cli drive +import --type docx --file ~/issue-hunter-loop/reports/YYYY-MM-DD-HHmm.md \
  --folder-token "$FOLDER_TOKEN" --name "Issue Hunter 报告 YYYY-MM-DD HH:mm" --as user
```

---

## Phase 7: Send Feishu Briefing

```bash
FEISHU_ID=$(python3 -c "import json; print(json.load(open('$HOME/issue-hunter-loop/config.json'))['feishu']['user_open_id'])")
lark-cli im +messages-send --as bot --user-id "$FEISHU_ID" --markdown $'简报内容'
```

---

## Phase 8: Implement Top 5

**From the 15 analyzed issues, select the top 5 by merge probability.** Then for each:

### 8.1 Read CONTRIBUTING.md FIRST
- Commit format (Conventional Commits, DCO sign-off, trailers)
- Test requirements (must pass before PR)
- CLA / contributor gate status
- Branch naming convention

### 8.2 Fork, Branch, Implement
```bash
gh repo fork <owner/repo> --clone --remote -- ~/issue-hunter-loop/repos/<repo-name>
cd ~/issue-hunter-loop/repos/<repo-name>
git checkout -b fix/<issue-number>-<short-description> upstream/main
# ... implement the fix ...
```

### 8.3 Self-Verify (MANDATORY)
- **Type check:** `tsgo --noEmit` / `mypy` / `pyright`
- **Lint:** `biome check` / `ruff` / `eslint`
- **Tests:** Run the repo's test suite. If the repo requires specific test commands (e.g. `scripts/run_tests.sh`), use those.
- **New tests:** Add regression test for the fix
- If any verification fails, fix before proceeding.

### 8.4 Commit (Follow Repo Convention)
```bash
git add <explicit paths, NEVER git add -A>
git commit -s -m "fix(scope): description

Details...

Fixes #N

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
Co-Authored-By: Happy <yesreply@happy.engineering>
"
```

### 8.5 Push to Personal Fork
```bash
git push origin <branch>
# NEVER push to upstream
# NEVER create a PR without user approval
```

---

## Phase 9: Generate Handoff Document

Write to `/tmp/issue-hunter-handoff-YYYY-MM-DD.md`:

```markdown
# Issue Hunter Handoff — YYYY-MM-DD

## Implemented (5 issues)

### 1. [repo] #N — title
- **Branch:** Xingkai98/<repo>:fix/N-short-desc
- **Issue:** <url>
- **Change:** <1-line summary>
- **Diff:** <brief description of changes>
- **Verification:** build ✓/✗, lint ✓/✗, tests ✓/✗
- **Contributing rules:** CLA needed? Commit format? Test requirements?
- **Risk:** low/medium/high

### 2. ...
...

## Analyzed (10 issues, not implemented)
### 6. [repo] #N — title
- **Score:** X/25
- **Merge probability:** reasoning
- **Why not implemented:** <lower merge prob / complex / CLA / already has PR>

### 7. ...
...

## Suggested Skills
- `review` — review the 5 implemented branches
```

---

## Phase 10: Update State

Add entries to `state.json` for all 15 analyzed issues + 5 implemented.

```json
{
  "issue_url": "...",
  "repo": "owner/repo",
  "issue_number": N,
  "status": "analyzed|implemented",
  "timestamp": "ISO-8601",
  "score": X,
  "summary": "...",
  "comment_assessment": {...}
}
```

---

## Phase 11: Update state.json
- Append all 15 issues (15 analyzed, 5 of which also implemented)
- Update `last_scan` timestamp
- Increment stats counters

---

## Safety Rules

- NEVER push to upstream repo — only to `origin` (personal fork)
- NEVER create a PR without user approval
- NEVER use `git add -A` — always commit by explicit path
- NEVER skip CONTRIBUTING.md — read it first
- NEVER claim tests pass if you didn't run them
- If anything seems off or risky, skip the implementation and note why in the handoff

## Durable Loop

Scheduled via CronCreate: daily at 4:07 AM, durable, recurring.
