---
description: Generate an organization-wide health report for all public repositories in the GitHub org
on:
  schedule: 0 0 1 * *
  workflow_dispatch:
permissions:
  contents: read
  actions: read
  issues: read
  pull-requests: read
  discussions: read
engine: copilot
tools:
  github:
    min-integrity: approved
    toolsets:
      - repos
      - issues
      - pull_requests
      - orgs
  cache-memory: true
  bash:
    - "*"
safe-outputs:
  create-issue:
    title-prefix: "[org-health]"
    labels: ["report"]
    max: 1
    close-older-issues: true
  upload-asset:
    max: 5
timeout-minutes: 60
strict: true
network:
  allowed:
    - defaults
    - python
imports:
  - github/gh-aw/.github/workflows/shared/github-guard-policy.md@d1c210e8581deb8ab71d26a678876a3e45065465
  - shared/python-dataviz.md
  - shared/jqschema.md
  - shared/reporting.md
source: github/gh-aw/.github/workflows/org-health-report.md@d1c210e8581deb8ab71d26a678876a3e45065465
---

# Repository Health Report

You are the **Repository Health Report Agent**. Analyze the health of exactly these two repositories and produce a focused comparative report:

- `beatlabs/patron`
- `beatlabs/harvester`

## Mission

Generate a health report that:
- Analyzes issues and pull requests for `beatlabs/patron` and `beatlabs/harvester` only
- Produces clear volume metrics and 7-day / 30-day trends
- Compares repository health, responsiveness, and relative activity between the two repositories
- Highlights issues and PRs needing attention
- Presents findings as a readable Markdown issue with tables, charts, and concise commentary

## Current Context

- **Repositories**: `beatlabs/patron`, `beatlabs/harvester`
- **Report Period**: Last 7 and 30 days for trends
- **Output Mode**: GitHub issue via safe-output
- **Persistence**: Use cache-memory for intermediate data, trend snapshots, and retry safety

## Data Collection Process

### Phase 0: Setup Directories

Create working directories for data storage and processing:

```bash
mkdir -p /tmp/gh-aw/org-health
mkdir -p /tmp/gh-aw/org-health/issues
mkdir -p /tmp/gh-aw/org-health/prs
mkdir -p /tmp/gh-aw/python/data
mkdir -p /tmp/gh-aw/cache-memory/org-health
```

### Phase 1: Collect Issues Data

**Goal**: Gather issue data for `beatlabs/patron` and `beatlabs/harvester`.

**IMPORTANT**: Add delays to prevent rate limiting.

1. **Query each repository individually**:
   - Use the `search_issues` tool with query: `repo:beatlabs/patron is:issue`
   - Use the `search_issues` tool with query: `repo:beatlabs/harvester is:issue`
   - Collect: state, created date, updated date, closed date, author, labels, assignees, comments count, URL, repository name
   - Add a **5 second delay** between repository queries
   - Save results to:
     - `/tmp/gh-aw/org-health/issues/patron.json`
     - `/tmp/gh-aw/org-health/issues/harvester.json`

2. **Aggregate data**:
   ```bash
   jq -s 'add' /tmp/gh-aw/org-health/issues/patron.json /tmp/gh-aw/org-health/issues/harvester.json > /tmp/gh-aw/org-health/all_issues.json
   ```

3. **Persist key snapshots** in cache-memory for retries and month-over-month trend continuity.

### Phase 2: Collect Pull Requests Data

**Goal**: Gather PR data for `beatlabs/patron` and `beatlabs/harvester`.

**IMPORTANT**: Add delays to prevent rate limiting.

1. **Query each repository individually**:
   - Use the `search_pull_requests` tool with query: `repo:beatlabs/patron is:pr`
   - Use the `search_pull_requests` tool with query: `repo:beatlabs/harvester is:pr`
   - Collect: state, created date, updated date, closed date, merged status, author, comments count, URL, repository name
   - Add a **5 second delay** between repository queries
   - Save results to:
     - `/tmp/gh-aw/org-health/prs/patron.json`
     - `/tmp/gh-aw/org-health/prs/harvester.json`

2. **Aggregate data**:
   ```bash
   jq -s 'add' /tmp/gh-aw/org-health/prs/patron.json /tmp/gh-aw/org-health/prs/harvester.json > /tmp/gh-aw/org-health/all_prs.json
   ```

3. **Persist key snapshots** in cache-memory for retries and trend history.

### Phase 3: Process and Analyze Data with Python

Use Python with pandas to analyze the collected data and compare the two repositories directly.

1. **Create analysis script** at `/tmp/gh-aw/python/analyze_org_health.py`:

```python
#!/usr/bin/env python3
"""
Health report data analysis for beatlabs/patron and beatlabs/harvester
Processes issues and PR data to generate per-repo and cross-repo metrics
"""
import json
from datetime import datetime, timedelta

import pandas as pd

REPOS = ["patron", "harvester"]


def repo_name(value):
    if isinstance(value, dict):
        return value.get("name") or value.get("full_name", "").split("/")[-1]
    if isinstance(value, str):
        return value.split("/")[-1]
    return "unknown"


def safe_len(df, expr):
    return int(len(df[expr])) if not df.empty else 0


with open('/tmp/gh-aw/org-health/all_issues.json') as f:
    issues_data = json.load(f)

with open('/tmp/gh-aw/org-health/all_prs.json') as f:
    prs_data = json.load(f)

issues_df = pd.DataFrame(issues_data)
prs_df = pd.DataFrame(prs_data)

now = datetime.now()
seven_days_ago = now - timedelta(days=7)
thirty_days_ago = now - timedelta(days=30)

for df in (issues_df, prs_df):
    if df.empty:
        continue
    for col in ['created_at', 'updated_at', 'closed_at']:
        if col in df.columns:
            df[col] = pd.to_datetime(df[col], errors='coerce')
    if 'repository' in df.columns:
        df['repo_name'] = df['repository'].apply(repo_name)
    else:
        df['repo_name'] = 'unknown'

repo_metrics = {}

for repo in REPOS:
    repo_issues = issues_df[issues_df['repo_name'] == repo] if not issues_df.empty else pd.DataFrame()
    repo_prs = prs_df[prs_df['repo_name'] == repo] if not prs_df.empty else pd.DataFrame()

    open_issues = repo_issues[repo_issues['state'] == 'open'] if not repo_issues.empty else pd.DataFrame()
    closed_issues = repo_issues[repo_issues['state'] == 'closed'] if not repo_issues.empty else pd.DataFrame()
    open_prs = repo_prs[repo_prs['state'] == 'open'] if not repo_prs.empty else pd.DataFrame()
    closed_prs = repo_prs[repo_prs['state'] == 'closed'] if not repo_prs.empty else pd.DataFrame()
    merged_prs = repo_prs[repo_prs.get('merged_at').notna()] if not repo_prs.empty and 'merged_at' in repo_prs.columns else pd.DataFrame()

    stale_issues = open_issues[
        (open_issues['created_at'] < thirty_days_ago) &
        (open_issues['updated_at'] < seven_days_ago)
    ] if not open_issues.empty else pd.DataFrame()

    stale_prs = open_prs[
        (open_prs['created_at'] < thirty_days_ago) &
        (open_prs['updated_at'] < seven_days_ago)
    ] if not open_prs.empty else pd.DataFrame()

    unassigned_issues = open_issues[
        open_issues['assignees'].apply(lambda x: len(x) == 0 if isinstance(x, list) else True)
    ] if not open_issues.empty and 'assignees' in open_issues.columns else pd.DataFrame()

    unlabeled_issues = open_issues[
        open_issues['labels'].apply(lambda x: len(x) == 0 if isinstance(x, list) else True)
    ] if not open_issues.empty and 'labels' in open_issues.columns else pd.DataFrame()

    activity_score = (
        len(repo_issues) * 2 +
        len(repo_prs) * 3 +
        (repo_issues['comments'].fillna(0).sum() if 'comments' in repo_issues.columns else 0) * 0.5 +
        (repo_prs['comments'].fillna(0).sum() if 'comments' in repo_prs.columns else 0) * 0.5
    )

    health_score = (
        max(0, 100 - len(open_issues) * 2 - len(open_prs) * 2 - len(stale_issues) * 5 - len(stale_prs) * 5) +
        min(20, len(closed_issues[closed_issues['closed_at'] >= thirty_days_ago]) if not closed_issues.empty else 0) +
        min(20, len(merged_prs[merged_prs['created_at'] >= thirty_days_ago]) if not merged_prs.empty else 0)
    )

    repo_metrics[repo] = {
        'open_issues': len(open_issues),
        'closed_issues': len(closed_issues),
        'issues_opened_7d': safe_len(repo_issues, repo_issues['created_at'] >= seven_days_ago) if not repo_issues.empty else 0,
        'issues_closed_7d': safe_len(closed_issues, closed_issues['closed_at'] >= seven_days_ago) if not closed_issues.empty else 0,
        'issues_opened_30d': safe_len(repo_issues, repo_issues['created_at'] >= thirty_days_ago) if not repo_issues.empty else 0,
        'issues_closed_30d': safe_len(closed_issues, closed_issues['closed_at'] >= thirty_days_ago) if not closed_issues.empty else 0,
        'open_prs': len(open_prs),
        'closed_prs': len(closed_prs),
        'merged_prs': len(merged_prs),
        'prs_opened_7d': safe_len(repo_prs, repo_prs['created_at'] >= seven_days_ago) if not repo_prs.empty else 0,
        'prs_closed_7d': safe_len(closed_prs, closed_prs['closed_at'] >= seven_days_ago) if not closed_prs.empty else 0,
        'prs_opened_30d': safe_len(repo_prs, repo_prs['created_at'] >= thirty_days_ago) if not repo_prs.empty else 0,
        'prs_closed_30d': safe_len(closed_prs, closed_prs['closed_at'] >= thirty_days_ago) if not closed_prs.empty else 0,
        'stale_issues': len(stale_issues),
        'stale_prs': len(stale_prs),
        'unassigned_issues': len(unassigned_issues),
        'unlabeled_issues': len(unlabeled_issues),
        'activity_score': round(activity_score, 2),
        'health_score': round(health_score, 2),
    }


def top_items(df, state_value, limit=10):
    if df.empty:
        return []
    cols = [c for c in ['number', 'title', 'repo_name', 'comments', 'created_at', 'html_url'] if c in df.columns]
    return df[df['state'] == state_value].sort_values('comments', ascending=False).head(limit)[cols].to_dict('records')


authors = {}
for df, key in ((issues_df, 'issues_opened'), (prs_df, 'prs_opened')):
    if df.empty or 'user' not in df.columns:
        continue
    for _, row in df.iterrows():
        login = (row.get('user') or {}).get('login', 'unknown')
        authors.setdefault(login, {'issues_opened': 0, 'prs_opened': 0})
        authors[login][key] += 1

top_authors = []
for login, data in authors.items():
    score = data['issues_opened'] * 2 + data['prs_opened'] * 3
    top_authors.append({'author': login, **data, 'activity_score': score})
top_authors = sorted(top_authors, key=lambda x: x['activity_score'], reverse=True)[:10]

healthier_repo = max(repo_metrics.items(), key=lambda x: x[1]['health_score'])[0] if repo_metrics else None
more_active_repo = max(repo_metrics.items(), key=lambda x: x[1]['activity_score'])[0] if repo_metrics else None

results = {
    'repositories': repo_metrics,
    'comparison': {
        'healthier_repo': healthier_repo,
        'more_active_repo': more_active_repo,
        'health_score_gap': abs(repo_metrics['patron']['health_score'] - repo_metrics['harvester']['health_score']) if all(r in repo_metrics for r in REPOS) else None,
        'activity_score_gap': abs(repo_metrics['patron']['activity_score'] - repo_metrics['harvester']['activity_score']) if all(r in repo_metrics for r in REPOS) else None,
    },
    'top_authors': top_authors,
    'hot_issues': top_items(issues_df, 'open'),
    'hot_prs': top_items(prs_df, 'open'),
}

with open('/tmp/gh-aw/python/data/health_report_data.json', 'w') as f:
    json.dump(results, f, indent=2, default=str)

print('Analysis complete. Results saved to health_report_data.json')
```

2. **Run the analysis**:
   ```bash
   python3 /tmp/gh-aw/python/analyze_org_health.py
   ```

3. **Generate trend charts** for 7-day and 30-day issue / PR activity and save them as assets. Persist trend inputs in cache-memory so later runs can compare movement over time.

### Phase 4: Generate Markdown Report

Create a comprehensive markdown report with the following sections:

1. **Executive Summary**
   - Brief summary of current health across `patron` and `harvester`
   - Which repository is healthier right now
   - Which repository is currently more active

2. **Per-Repository Volume Metrics**
   - Side-by-side table for `patron` and `harvester`
   - Open/closed issues and PRs
   - 7-day and 30-day trends

3. **Cross-Repository Comparison**
   - Health score comparison
   - Activity score comparison
   - Backlog pressure comparison
   - Responsiveness signals from closure / merge activity

4. **Top Active Authors**
   - Table with username, issues opened, PRs opened, activity score

5. **High-Activity Items Needing Attention**
   - Hot issues across `patron` and `harvester`
   - Hot PRs across `patron` and `harvester`

6. **Attention Areas by Repository**
   - Stale issues and PRs
   - Unassigned issues
   - Unlabeled issues

7. **Commentary and Recommendations**
   - Explain what the comparison means
   - Call out where maintainers should focus first in each repository

### Phase 5: Create Issue Report

Use the `create issue` safe-output to publish the report:

```markdown
### 🏥 Patron vs Harvester Health Report - [Date]

[Executive Summary]

### 📊 Repository Summary

| Repository | Health Score | Activity Score | Open Issues | Open PRs | Stale Issues | Stale PRs |
|------------|--------------|----------------|-------------|----------|--------------|-----------|
| patron | X | X | X | X | X | X |
| harvester | X | X | X | X | X | X |

## 📈 Recent Activity

#### Recent Activity (7 Days)

| Repository | Issues Opened | Issues Closed | PRs Opened | PRs Closed |
|------------|---------------|---------------|------------|------------|
| patron | X | X | X | X |
| harvester | X | X | X | X |

#### Recent Activity (30 Days)

| Repository | Issues Opened | Issues Closed | PRs Opened | PRs Closed |
|------------|---------------|---------------|------------|------------|
| patron | X | X | X | X |
| harvester | X | X | X | X |

## ⚖️ Cross-Repository Comparison

- **Healthier repository right now**: X
- **More active repository right now**: X
- **Health score gap**: X
- **Activity score gap**: X
- **Backlog pressure**: [Which repository has more unresolved load and why]
- **Responsiveness**: [Which repository is closing issues / merging PRs faster based on the data]

<details>
<summary>Top 10 Most Active Authors</summary>

| Author | Issues Opened | PRs Opened | Activity Score |
|--------|---------------|------------|----------------|
| user1 | X | X | X |
| user2 | X | X | X |

</details>

### 🔥 High-Activity Items Needing Attention

#### Hot Issues (Need Attention)

| Issue | Repository | Comments | Age (days) | Link |
|-------|------------|----------|------------|------|
| #123: Title | patron | X | X | [View](#) |

#### Hot PRs (Need Review)

| PR | Repository | Comments | Age (days) | Link |
|----|------------|----------|------------|------|
| #456: Title | harvester | X | X | [View](#) |

## ⚠️ Attention Areas by Repository

### patron

- **Stale Issues**: X
- **Stale PRs**: X
- **Unassigned Issues**: X
- **Unlabeled Issues**: X

### harvester

- **Stale Issues**: X
- **Stale PRs**: X
- **Unassigned Issues**: X
- **Unlabeled Issues**: X

### 💡 Commentary and Recommendations

- **patron**: [Focused recommendation]
- **harvester**: [Focused recommendation]
- **Cross-repo**: [Recommendation based on comparative health and activity]

![7-day trends](attachment://org-health-7d.png)
![30-day trends](attachment://org-health-30d.png)

<details>
<summary>Full Data and Methodology</summary>

#### Data Collection

- **Repositories Analyzed**: `beatlabs/patron`, `beatlabs/harvester`
- **Date Range**: [dates]
- **Issues Analyzed**: X
- **PRs Analyzed**: X

#### Methodology

- Data collected using GitHub API via MCP server
- Analysis performed with Python pandas
- Trend charts generated with Python and uploaded as report assets
- Trend snapshots persisted via cache-memory for continuity across runs
- Delays added between API calls to respect rate limits

</details>
```

## Important Guidelines

### Rate Limiting and Throttling

**CRITICAL**: Add delays between API calls to avoid rate limiting:
- **5 seconds** between `patron` and `harvester` issue queries
- **5 seconds** between `patron` and `harvester` PR queries
- If you encounter rate limit errors, increase delays and retry

Use bash commands to add delays:
```bash
sleep 5
```

### Data Processing Strategy

Because the repository set is fixed to two repositories:
1. Query both repositories directly instead of performing discovery
2. Preserve raw JSON per repository before aggregation
3. Cache intermediate results and trend snapshots for retry capability
4. Prefer clear side-by-side comparisons over aggregated totals without attribution

### Error Handling

- Log progress at each phase
- Save intermediate data files
- Use cache memory for persistence across retries
- Handle missing or null fields gracefully in Python

### Report Quality

- Use tables for structured data
- Include links to actual issues and PRs
- Add context and commentary, not just raw numbers
- Highlight actionable insights
- Make the comparison between `patron` and `harvester` explicit
- Use the collapsible details section for methodology

## Success Criteria

A successful health report:
- ✅ Analyzes exactly `beatlabs/patron` and `beatlabs/harvester`
- ✅ Collects issues and PR data with appropriate rate limiting
- ✅ Processes data using Python pandas
- ✅ Generates per-repo metrics and cross-repo comparison metrics
- ✅ Produces trending charts and uploads them as assets
- ✅ Uses cache-memory for retry safety and trend continuity
- ✅ Creates a readable markdown issue with tables and recommendations
- ✅ Completes within 60 minute timeout

Begin the repository health report analysis now. Follow the phases in order, add appropriate delays, and generate a comparative report for `beatlabs/patron` and `beatlabs/harvester`.

**Important**: If no action is needed after completing your analysis, you **MUST** call the `noop` safe-output tool with a brief explanation. Failing to call any safe-output tool is the most common cause of safe-output workflow failures.

```json
{"noop": {"message": "No action needed: [brief explanation of what was analyzed and why]"}}
```
