---
description: Stable schema for daily metrics collected for patron and harvester.
---

# Organization Metrics Schema

This schema defines the stable JSON contract for daily metrics collected from `beatlabs/patron` and `beatlabs/harvester`.

Downstream consumers such as the audit workflow and weekly summary depend on these keys remaining stable. Do not rename, remove, or repurpose existing keys. Only add new keys in a backward-compatible way.

## Top-level object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `collection_timestamp` | `string` | Yes | ISO 8601 timestamp for when collection completed. |
| `repos` | `array of objects` | Yes | Per-repository daily metrics. Current producers emit one entry for `beatlabs/patron` and one entry for `beatlabs/harvester`. |
| `cross_repo_summary` | `object` | Yes | Aggregated summary across all repositories in `repos`. |

## `repos[]` object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `repo` | `string` | Yes | Repository identifier. Allowed values for the current producer are `beatlabs/patron` and `beatlabs/harvester`. |
| `open_issues` | `integer` | Yes | Current count of open issues. |
| `closed_issues_24h` | `integer` | Yes | Count of issues closed during the last 24 hours. |
| `open_prs` | `integer` | Yes | Current count of open pull requests. |
| `merged_prs_24h` | `integer` | Yes | Count of pull requests merged during the last 24 hours. |
| `pending_reviews` | `integer` | Yes | Count of open pull requests currently waiting on review. |
| `ci_pass_rate_7d` | `number \| null` | Yes | CI pass rate over the last 7 days, represented as a decimal ratio from `0` to `1`. Use `null` when no qualifying runs exist. |
| `ci_avg_duration_minutes` | `number \| null` | Yes | Average CI run duration over the last 7 days in minutes. Use `null` when no qualifying runs exist. |
| `agentic_workflow_runs_24h` | `integer` | Yes | Count of agentic workflow runs during the last 24 hours. |
| `agentic_workflow_success_rate` | `number \| null` | Yes | Agentic workflow success rate during the last 24 hours, represented as a decimal ratio from `0` to `1`. Use `null` when no qualifying runs exist. |

## `cross_repo_summary` object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `repo_count` | `integer` | Yes | Number of repository entries included in `repos`. |
| `total_open_issues` | `integer` | Yes | Sum of `open_issues` across all repositories. |
| `total_closed_issues_24h` | `integer` | Yes | Sum of `closed_issues_24h` across all repositories. |
| `total_open_prs` | `integer` | Yes | Sum of `open_prs` across all repositories. |
| `total_merged_prs_24h` | `integer` | Yes | Sum of `merged_prs_24h` across all repositories. |
| `total_pending_reviews` | `integer` | Yes | Sum of `pending_reviews` across all repositories. |
| `avg_ci_pass_rate_7d` | `number \| null` | Yes | Average of non-null `ci_pass_rate_7d` values across repositories. |
| `avg_ci_duration_minutes` | `number \| null` | Yes | Average of non-null `ci_avg_duration_minutes` values across repositories. |
| `total_agentic_workflow_runs_24h` | `integer` | Yes | Sum of `agentic_workflow_runs_24h` across all repositories. |
| `avg_agentic_workflow_success_rate` | `number \| null` | Yes | Average of non-null `agentic_workflow_success_rate` values across repositories. |

## JSON shape

```json
{
  "collection_timestamp": "2026-04-13T01:00:00Z",
  "repos": [
    {
      "repo": "beatlabs/patron",
      "open_issues": 0,
      "closed_issues_24h": 0,
      "open_prs": 0,
      "merged_prs_24h": 0,
      "pending_reviews": 0,
      "ci_pass_rate_7d": 1,
      "ci_avg_duration_minutes": 4.2,
      "agentic_workflow_runs_24h": 0,
      "agentic_workflow_success_rate": null
    },
    {
      "repo": "beatlabs/harvester",
      "open_issues": 0,
      "closed_issues_24h": 0,
      "open_prs": 0,
      "merged_prs_24h": 0,
      "pending_reviews": 0,
      "ci_pass_rate_7d": 0.85,
      "ci_avg_duration_minutes": 6.1,
      "agentic_workflow_runs_24h": 0,
      "agentic_workflow_success_rate": null
    }
  ],
  "cross_repo_summary": {
    "repo_count": 2,
    "total_open_issues": 0,
    "total_closed_issues_24h": 0,
    "total_open_prs": 0,
    "total_merged_prs_24h": 0,
    "total_pending_reviews": 0,
    "avg_ci_pass_rate_7d": 0.925,
    "avg_ci_duration_minutes": 5.15,
    "total_agentic_workflow_runs_24h": 0,
    "avg_agentic_workflow_success_rate": null
  }
}
```

## Stability rules

- Keep top-level keys stable: `collection_timestamp`, `repos`, `cross_repo_summary`.
- Keep per-repo metric keys stable for all downstream consumers.
- Preserve numeric semantics: counts are integers, rates are decimal ratios, durations are minutes.
- Use `null` instead of inventing placeholder values when data is empty or unavailable.
