# Issue Hunter Loop — Research & Analysis

Research-only automation: discover trending AI/Agent/LLM projects, find 6 high-quality issues (3 from priority repos, 3 from open exploration), analyze background and design fix strategies, produce a report.

## Configuration

- `config.json` — Feishu user open_id (gitignored, see `config.example.json`)
- `repos.json` — priority repo list, editable (tracked, see `repos.example.json` for format)

## Core Rules

- **Research only** — do NOT fork, clone, fix, or push code
- Work inside `~/issue-hunter-loop/`
- Read priority repos from `repos.json` at runtime — if empty or yields nothing useful, fill all 6 slots from exploration
- Track all processed issues in `state.json` — check it before selecting candidates, NEVER re-process an issue. `state.json` is the structural dedup mechanism. Every analyzed issue goes in, no exceptions.
- One report per loop iteration, **always 6 issues**
- Output report to `reports/YYYY-MM-DD-HHmm.md`

## Workflow

### 0. Cleanup

Remove repos from `repos/` whose PRs are merged/closed (keep workspace lean).

```bash
du -sh ~/issue-hunter-loop/repos/*/ | sort -rh
# rm -rf ~/issue-hunter-loop/repos/<stale-repo>
```

### 1. Multi-Source Discovery

Use **four sources** in parallel, then merge and deduplicate:

#### 1a. Priority Repos (scan first)

Read the priority repo list from `repos.json`. If the file is empty `[]` or missing, skip this step and source all 6 candidates from exploration (1b–1d).

```bash
# Read priority repos from config
REPOS=$(python3 -c "import json; print(' '.join(r['repo'] for r in json.load(open('$HOME/issue-hunter-loop/repos.json'))))" 2>/dev/null || echo "")

if [ -n "$REPOS" ]; then
  for repo in $REPOS; do
    gh api "repos/$repo/issues?labels=bug&state=open&per_page=10&sort=updated" \
      --jq '.[] | "#\(.number) [👍\(.reactions.total_count)] [\(.updated_at[:10])] \(.title)"' &
    gh api "repos/$repo/issues?labels=enhancement&state=open&per_page=5&sort=updated" \
      --jq '.[] | "#\(.number) [👍\(.reactions.total_count)] [\(.updated_at[:10])] \(.title)"' &
  done
  wait
fi
```

**Dynamic expansion**: periodically discover new fast-growing AI/Agent/LLM repos (stars > 10k, growth > 1k/week, language Go/Python/Rust/TypeScript, topic `agent`/`llm`/`ai`). When found, append to `repos.json`.

#### 1b. GitSense — automated issue discovery

```bash
export PATH="$HOME/.local/bin:$PATH"

# Find issues matching AI/agent skills in repos with >500 stars
gitsense find --skills python,typescript,rust,go,llm,ai-agent --stars 500 \
  --labels bug,enhancement,help+wanted --updated-days 30 --limit 20 --no-llm

# Evaluate repo quality before diving in
gitsense radar --skills python,typescript,llm --days 90 --sample 20
```

#### 1c. GitHub Trending — discover hot repos

```bash
# Trending today (via gh api)
gh api search/repositories --jq '.items[:15] | .[] | "\(.full_name) [⭐\(.stargazers_count)] [\(.pushed_at[:10])] \(.description[:120])"' \
  -f q='stars:>1000 pushed:>2026-06-01 topic:ai topic:agent topic:llm' \
  -f sort=stars -f order=desc

# Recently created, fast-growing AI repos
gh api search/repositories --jq '.items[:10] | .[] | "\(.full_name) [⭐\(.stargazers_count)] [created:\(.created_at[:10])] \(.description[:120])"' \
  -f q='created:>2026-01-01 stars:>5000 topic:ai' \
  -f sort=stars -f order=desc
```

#### 1d. Global search — bugs with community interest

```bash
# Bugs with comments and reactions (indicates maintainer/user engagement)
gh search issues --label bug --state open --sort updated --limit 30 \
  --json number,title,url,repository,commentsCount,updatedAt

# Bugs + help wanted in quality repos
gh search issues --label "help wanted" --label bug --state open --sort updated --limit 20 \
  --json number,title,url,repository,commentsCount,updatedAt
```

### 2. Filter & Score Candidates

**Must-pass filters:**
- Stars > 100 (or well-known org like `rust-lang/`, `microsoft/`, etc.)
- Pushed within last 30 days
- Language: Go, Python, Rust, TypeScript
- NOT a hacktoberfest/spam/template repo
- NOT already in `state.json` (check by `issue_url` — exact match or same repo+number)

**Scoring dimensions** (1-5 each, total /25):

| Dimension | 5 points | 1 point |
|-----------|----------|---------|
| **Technical depth** | Real bug with repro steps, stack traces, error messages | Vague description, no details |
| **Maintainer engagement** | OWNER/MEMBER has commented | Zero comments, no engagement |
| **Community interest** | 3+ 👍 reactions, multiple users confirming | Zero reactions |
| **Scope clarity** | Clear acceptance criteria, specific files/hints | "Refactor entire X", no boundaries |
| **Analysis value** | Can write an insightful fix design | Trivial or purely config/typo |

**Red flags (auto-skip):**
- Already has an active assignee or linked PR
- `/claim` comments from active contributors
- Auth/security, database migrations, payment logic
- Pure data entry, typo fixes, config changes
- `hacktoberfest` or `good first issue` labels (too competitive, too shallow)

### 3. Community Status Assessment (MANDATORY)

**Before final selection**, read the full comment thread of each shortlisted candidate. Assess:

#### A. Implementation status
- Is there an existing PR linked in the comments? If so:
  - What's its status? (open/closed/merged/draft)
  - What's the quality? (passes CI? reviewed? maintainer feedback?)
  - If a good implementation already exists → **demote heavily or skip**. The issue is already handled.
- Are there comments like "I'm working on this" or "taking this" from active contributors? → **skip if claimed**.

#### B. Timeliness
- When was the last meaningful comment (not stale bot, not +1)?
- If last activity > 60 days ago → **-2 to score** (may be stale or low priority)
- If last activity < 7 days ago → **+1 to score** (hot topic)
- Is the issue still reproducible on the latest version? Check for "still broken on vX.Y.Z" comments.

#### C. Community viability
- Are there multiple users confirming the same issue? → **strong signal**
- Is there maintainer pushback? ("this is by design", "won't fix") → **skip**
- Is the discussion constructive (debugging, narrowing down) or noise ("+1", "me too")?
- Has a maintainer said "PR welcome" or "lgtm"? → **strong signal**

**Adjust scores** based on this assessment before selecting the final 3.

### 4. Select Top 6

Select **6 issues total**, split across two tiers:

**Tier A — 3 from priority repos** (if available):
- Pick the 3 highest-scoring issues discovered from `repos.json`
- If priority repos are empty or yield < 3 viable candidates, fill remaining slots from exploration

**Tier B — 3 from open exploration** (trending, GitSense, global search):
- Pick the 3 highest-scoring issues from all non-priority sources
- Prioritize recently trending repos, fast-growing projects, and unexpected discoveries

**Diversity rules** (apply across all 6):
- No more than 2 issues from the same repo
- At least 2 different languages across the set
- Mix of bug fixes and well-scoped enhancements

Prefer:
1. **Real bugs over feature requests** (more insight value)
2. **Active community** (comments, reactions, recent activity)
3. **Diverse technologies** (not all TypeScript, not all Python)
4. **No existing quality implementation** (don't duplicate work)

### 5. Deep Analysis — For Each Issue

For each of the 3 selected issues, analyze:

#### A. Background
- What is this project? Stars, language, purpose
- What subsystem/component is affected?
- Has the maintainer engaged? What did they say?
- Are there related issues or PRs?

#### B. Problem or Requirement
- What exactly is broken or missing?
- Can we reproduce it just by reading the issue + code?
- What's the user impact? (who's affected, how badly)
- Edge cases or constraints mentioned

#### C. Fix Strategy Design
- **Root cause analysis**: trace through code if possible (read relevant files)
- **Proposed approach**: what to change, at a high level
- **Files to modify**: list specific paths with line ranges
- **Risk assessment**: what could go wrong, what tests are needed
- **Alternative approaches**: if there are multiple ways, compare them

Use `gh issue view` and `gh api` to read issue bodies and comments. If the fix strategy requires reading actual source code, use `gh api "repos/<owner>/<repo>/contents/<path>"` to fetch files without cloning.

### 6. Write Report

Write to `~/issue-hunter-loop/reports/YYYY-MM-DD-HHmm.md`:

```markdown
# Issue Hunter Report — YYYY-MM-DD HH:mm

## Sources scanned
- Priority repos: X issues found
- GitSense: X candidates
- GitHub Trending: X new repos discovered
- Global search: X results

---

## Issue 1: [repo] #N — [title]

**Link:** [url]
**Score:** X/20 (depth: X, engagement: X, interest: X, clarity: X, value: X)

### Background
...

### Problem
...

### Fix Strategy
...

---

## Issue 2: [repo] #N — [title]

...

---

## Issue 3: [repo] #N — [title]

...

---

## Summary
- Total candidates scanned: X
- Final 3 selection rationale: ...
- New repos added to watchlist: ...
```

### 7. Upload to Feishu Drive

Create a Markdown file in the Feishu Drive folder `Issue Hunter Reports`:

```bash
# Create folder if not exists (use lark-drive skill)
# Upload report markdown (use lark-markdown skill)
```

### 8. Send Briefing via Feishu

Send a concise text summary to the user via Feishu DM:

```bash
FEISHU_USER_ID=$(cat ~/issue-hunter-loop/config.json | python3 -c "import sys,json; print(json.load(sys.stdin)['feishu']['user_open_id'])")
lark-cli im +messages-send --as bot --user-id "$FEISHU_USER_ID" \
  --markdown $'## 🎯 Issue Hunter 简报 — YYYY-MM-DD\n\n**Issue 1:** [repo] #N — title\n- 背景：...\n- 方案：...\n\n**Issue 2:** ...\n\n**Issue 3:** ...\n\n📄 完整报告：`reports/YYYY-MM-DD-HHmm.md`'
```

### 9. Update State

Add entries to `state.json` for all 6 issues:

```json
{
  "issue_url": "https://github.com/owner/repo/issues/N",
  "repo": "owner/repo",
  "issue_number": N,
  "status": "analyzed|skipped",
  "timestamp": "ISO-8601",
  "score": X,
  "summary": "one-line summary",
  "comment_assessment": {
    "last_activity": "ISO-8601",
    "has_existing_pr": false,
    "pr_quality": "none|draft|in-review|ready",
    "maintainer_stance": "lgtm|pr-welcome|no-response|wont-fix",
    "skip_reason": null
  }
}
```

## Quality Standards

- Issues must have **real technical substance** — not typo fixes or trivial config
- Fix strategies must be **actionable** — someone should be able to implement from this analysis
- Reports must cite **specific code paths and file names** where possible
- **No PRs, no forks, no clones, no pushing code**

## Durable Loop

Scheduled via CronCreate: daily at 4:00 AM, durable, recurring.
