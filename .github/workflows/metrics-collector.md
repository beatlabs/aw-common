---
description: Collect daily metrics for patron and harvester and publish structured outputs.
on:
  schedule:
    - cron: "0 1 * * *"
  workflow_dispatch:
permissions:
  contents: read
  actions: read
  issues: read
  pull-requests: read
tools: github[repos, issues, pull_requests, actions, search], bash, cache-memory
safe-outputs:
  upload-asset:
    max: 10
  messages:
    max: 1
imports:
  - shared/mood.md
  - shared/python-dataviz.md
  - shared/trending-charts-simple.md
  - shared/jqschema.md
  - shared/org-metrics-schema.md
engine: copilot
strict: true
timeout-minutes: 45
network:
  allowed: [defaults, python]
---

# Metrics Collector

You are the daily metrics collector for `beatlabs/patron` and `beatlabs/harvester`.

## Goal

Collect daily repository health metrics for exactly these two repositories:

- `beatlabs/patron`
- `beatlabs/harvester`

Produce structured JSON that conforms exactly to the schema defined in `shared/org-metrics-schema.md`.

## Requirements

1. Create working directories before collecting data:

   ```bash
   mkdir -p /tmp/gh-aw/metrics
   mkdir -p /tmp/gh-aw/metrics/charts
   ```

2. For each repository, gather the following metrics:

   - Issue counts:
     - `open_issues`
     - `closed_issues_24h`
   - Pull request counts:
     - `open_prs`
     - `merged_prs_24h`
     - `pending_reviews`
   - CI health over the last 7 days:
     - `ci_pass_rate_7d`
     - `ci_avg_duration_minutes`
   - Agentic workflow activity over the last 24 hours:
     - `agentic_workflow_runs_24h`
     - `agentic_workflow_success_rate`

3. Add a 5 second delay between repository query groups to reduce rate-limit pressure.

4. Handle empty or missing data gracefully:

   - Use `0` for count fields when no matching records exist.
   - Use `null` for rate or duration fields when there is insufficient data.
   - Do not fail the workflow solely because a specific metric source is empty.

5. Build a `cross_repo_summary` object from the per-repo metrics using the shared schema.

6. Save the final JSON payload to:

   ```text
   /tmp/gh-aw/metrics/daily-YYYY-MM-DD.json
   ```

   Use the UTC date for `YYYY-MM-DD`.

7. Validate the JSON structure against the shared schema contract before upload.

8. Generate trending charts in Python comparing `beatlabs/patron` and `beatlabs/harvester` using the collected daily metrics and any available cached history.

   Recommended charts:

   - issues and PRs comparison
   - CI pass rate comparison
   - workflow activity comparison

   Save charts under `/tmp/gh-aw/metrics/charts/`.

9. Upload outputs via `upload-asset`:

   - the daily JSON file
   - generated chart images

10. Store trend data in `cache-memory` so future runs can support month-over-month comparison.

## Collection guidance

- Use GitHub repository, issue, pull request, actions, and search capabilities only.
- Restrict all collection and analysis to `beatlabs/patron` and `beatlabs/harvester`.
- For pending reviews, count open pull requests that are waiting on reviewer action.
- For CI metrics, use the last 7 days of relevant workflow runs to compute pass rate and average duration.
- For agentic workflow metrics, identify relevant workflow runs in the last 24 hours and compute total runs plus success rate.
- Keep output keys stable because downstream audit and weekly summary workflows depend on them.

## Expected output behavior

- Emit one final structured message summarizing collection success, saved file path, uploaded assets, and any graceful fallbacks used.
- Ensure the JSON artifact remains the source of truth for downstream consumers.
