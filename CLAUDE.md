# Issue Hunter Loop — Research & Analysis

Research-only automation: discover trending AI/Agent/LLM projects, find 3 high-quality issues, analyze background and design fix strategies, produce a report.

## Core Rules

- **Research only** — do NOT fork, clone, fix, or push code
- Work inside `~/issue-hunter-loop/`
- Track all processed issues in `state.json` — NEVER re-process an issue
- One report per loop iteration, **always 3 issues**
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

Scan these repos for open bugs and enhancements with maintainer participation:

| Repo | Stars | Language | Focus |
|------|-------|----------|-------|
| `HKUDS/nanobot` | 44k | Python | AI agent framework |
| `jingyaogong/minimind` | 52k | Python | LLM training/inference |
| `karpathy/nanochat` | 55k | Python | LLM chat from scratch |
| `openclaw/openclaw` | 380k | TypeScript | AI agent runtime |
| `NousResearch/hermes-agent` | 203k | Python | AI agent framework |
| `earendil-works/pi` | 66k | TypeScript | Agent toolkit (LLM API, agent loop, TUI, CLI) |

**Dynamic expansion**: periodically discover new fast-growing AI/Agent/LLM repos (stars > 10k, growth > 1k/week, language Go/Python/Rust/TypeScript, topic `agent`/`llm`/`ai`). Add to this table when found.

```bash
for repo in HKUDS/nanobot jingyaogong/minimind karpathy/nanochat openclaw/openclaw NousResearch/hermes-agent earendil-works/pi; do
  gh api "repos/$repo/issues?labels=bug&state=open&per_page=10&sort=updated" \
    --jq '.[] | "#\(.number) [👍\(.reactions.total_count)] [\(.updated_at[:10])] \(.title)"' &
  gh api "repos/$repo/issues?labels=enhancement&state=open&per_page=5&sort=updated" \
    --jq '.[] | "#\(.number) [👍\(.reactions.total_count)] [\(.updated_at[:10])] \(.title)"' &
done
wait
```

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
- NOT already in `state.json`

**Scoring dimensions** (1-5 each, total /20):

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

### 3. Select Top 3

From the filtered list, pick the 3 highest-scoring issues across **diverse repos**. Avoid picking 2+ from the same repo. Prefer:

1. **Real bugs over feature requests** (more insight value)
2. **Active community** (comments, reactions, recent activity)
3. **Diverse technologies** (not all TypeScript, not all Python)

### 4. Deep Analysis — For Each Issue

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

### 5. Write Report

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

### 6. Upload to Feishu Drive

Create a Markdown file in the Feishu Drive folder `Issue Hunter Reports`:

```bash
# Create folder if not exists (use lark-drive skill)
# Upload report markdown (use lark-markdown skill)
```

### 7. Send Briefing via Feishu

Send a concise text summary to the user via Feishu DM:

```bash
lark-cli im +messages-send --as bot --user-id ou_6a162aa9bad0f5860f74829757e6dad5 \
  --markdown $'## 🎯 Issue Hunter 简报 — YYYY-MM-DD\n\n**Issue 1:** [repo] #N — title\n- 背景：...\n- 方案：...\n\n**Issue 2:** ...\n\n**Issue 3:** ...\n\n📄 完整报告：`reports/YYYY-MM-DD-HHmm.md`'
```

### 8. Update State

Add entries to `state.json` for all 3 issues:

```json
{
  "issue_url": "https://github.com/owner/repo/issues/N",
  "repo": "owner/repo",
  "issue_number": N,
  "status": "analyzed",
  "timestamp": "ISO-8601",
  "summary": "one-line summary"
}
```

## Quality Standards

- Issues must have **real technical substance** — not typo fixes or trivial config
- Fix strategies must be **actionable** — someone should be able to implement from this analysis
- Reports must cite **specific code paths and file names** where possible
- **No PRs, no forks, no clones, no pushing code**

## Durable Loop

Scheduled via CronCreate: daily at 4:00 AM, durable, recurring.
