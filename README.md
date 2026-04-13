# aw-common

Common agentic workflows for [gh-aw](https://github.github.com/gh-aw/introduction/overview/) — GitHub's framework for AI-powered workflow automation.

## Structure

Each workflow consists of:

- **`.md`** — The workflow definition: frontmatter (triggers, permissions, tools, imports) + agent prompt
- **`.lock.yml`** — Auto-generated GitHub Actions file produced by `gh aw compile`. Do not edit manually.

Shared prompt fragments live in `shared/` and are pulled in via the `imports:` frontmatter key.

## Workflows

| Workflow | Schedule | Description |
|----------|----------|-------------|
| `org-health-report` | Monthly (1st) | Analyzes issues and PRs for patron and harvester, produces comparative health metrics and an issue report |
| `metrics-collector` | Daily (1am UTC) | Collects CI, issue, PR, and dependency metrics for patron and harvester, produces JSON + charts |
| `audit-workflows` | Weekly (Mon 2am UTC) | Audits agentic workflow health across aw-common, patron, and harvester — classifies as healthy/degraded/broken |
| `stale-repo-identifier` | Monthly (1st) | Scans org repositories for staleness signals and files issues for inactive repos |
| `ubuntu-image-analyzer` | Monthly (1st) | Analyzes the default Ubuntu Actions runner image and maintains Docker mimic documentation |

## Shared Imports

| Import | Purpose |
|--------|---------|
| `shared/mood.md` | Agent personality/tone |
| `shared/python-dataviz.md` | Python environment setup with NumPy, Pandas, Matplotlib, Seaborn, SciPy + artifact upload |
| `shared/jqschema.md` | `jq` utilities and schema inference tooling |
| `shared/reporting.md` | Report formatting guidelines (header levels, progressive disclosure) |
| `shared/trending-charts-simple.md` | Python chart environment with cache-memory integration for trending data |
| `shared/go-ci.md` | Go CI context: Taskfile targets, linting config, test commands, network allowlist for Go module proxies |
| `shared/go-security.md` | Go security scanning guidance: govulncheck, gosec, dependency audit, CVE reporting |
| `shared/issue-triage-go.md` | Go-specific issue triage taxonomy: component classification, priority rules, label scheme |
| `shared/semver-policy.md` | Semantic versioning policy for Go libraries: breaking change definitions, severity classification, labeling rules |
| `shared/org-metrics-schema.md` | Stable JSON schema for daily metrics collection: per-repo CI/issue/PR/dependency fields, cross-repo summary |
| `shared/release-notes-template.md` | Tag-based release notes template: Breaking Changes, Features, Improvements, Bug Fixes, Dependencies, Contributors |

## Usage

Workflows are compiled with:

```sh
gh aw compile
```

This reads the `.md` definitions and generates the corresponding `.lock.yml` files. The `.gitattributes` marks lock files as `linguist-generated` and uses `merge=ours` to avoid merge conflicts on generated content.
