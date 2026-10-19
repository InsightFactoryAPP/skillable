---
name: fleet-health-dashboard
description: >-
  List every GitHub repo the authenticated user owns or controls — including
  every repo in organizations they own — collect each repo's health signals in
  parallel (open Dependabot/Renovate PRs, unassigned and stale PRs, Dependabot
  alerts by severity, default-branch CI status, pinned Node version, last push),
  and publish them as one live dashboard whose rows live in a shared database so
  later refreshes update data instead of rebuilding the page. Each row suggests
  the fleet skill that fixes it. Can be refreshed on a schedule. Use whenever the
  user wants an overview, status board, or morning dashboard of all their repos,
  to see which repos need attention, to track security alerts and bot PRs across
  orgs, or to refresh an existing fleet dashboard. Also triggers on
  /fleet-health-dashboard with optional refresh, schedule, or an org filter.
argument-hint: "[refresh|schedule] [org=<name>]"
---

# Fleet Health Dashboard

Goal: one page that answers "which of my repos need me today, and which skill fixes it?" across every repo the user owns or controls. The fleet skills in this collection *act*; this one *shows* — and points each problem at the skill that acts on it.

## When to use

- "Show me the state of all my repos" / "morning dashboard" / "what needs attention?"
- "How many open Dependabot alerts / bot PRs do I have across my orgs?"
- "Refresh the fleet dashboard" / "update it every weekday morning"
- `/fleet-health-dashboard [refresh|schedule] [org=<name>]`
- Not for fixing anything — hand off to the suggested skill instead.

## 0. Mode

Parse `$ARGUMENTS` / the user message:

- **(default)** — build or rebuild the dashboard.
- **`refresh`** — re-collect and update the existing dashboard's data in place.
- **`schedule`** — set up a recurring refresh (step 6).
- **Scope filter** — `org=<name>`, "only public", "skip <org>": a filter on the repo union.

If `gh` is not authenticated, stop and tell the user to run `gh auth login`. A 403 on org memberships means the token lacks `read:org` — tell them to run `gh auth refresh -s read:org`.

## 1. List repos they own or control

Use the **same three-source union as `sweep-all-dependency-prs`**: personal repos (`gh repo list <user> --limit 1000 --no-archived --source`), every repo in orgs where the membership is `state == "active"` and `role == "admin"` (`/user/memberships/orgs`), and other repos with `permissions.admin` or `permissions.maintain` from `/user/repos?affiliation=collaborator,organization_member`. Paginate to completion, dedup by `full_name`, skip forks, archived, and disabled repos unless asked.

## 2. Collect signals in parallel

Per repo, in parallel batches of ~10–15 (on a `secondary rate limit` 403, sleep ~60 s and retry with a smaller batch). Prefer one GraphQL query per batch for PR and repo fields, and REST for alerts and Actions. The exact queries and the field list are in [references/data-model.md](references/data-model.md).

| Signal | Source |
| --- | --- |
| Open bot PRs (Dependabot, Renovate), how many green / red / conflicting | PRs + status rollup |
| Unassigned PRs by others (no assignee, reviewer, or review) | PRs |
| Stale PRs (no activity > 2 days) | PRs `updatedAt` |
| Open Dependabot alerts by severity | `dependabot/alerts?state=open` (403 → "n/a") |
| Default-branch CI status and when it last went red | latest workflow runs on the default branch |
| Pinned Node version (`.nvmrc`, `engines`, `setup-node`) vs. current LTS | file contents |
| Last push, open issue count, visibility, primary language | repo fields |

A failing probe marks that cell `n/a` with the reason; it never aborts the sweep.

Compute a **status** per repo — `🔴 needs you` (critical/high alert, red default branch, or a red bot PR), `🟡 attention` (stale or unassigned PRs, moderate alerts, Node behind LTS), `🟢 healthy` — and a **next action** that names the skill:

| Condition | Suggested skill |
| --- | --- |
| Bot PRs open | `/sweep-dependency-prs <repo>` |
| Critical/high Dependabot alerts | `/fix-dependabot-alert <alert-url>` |
| Default branch red | `/fix-failing-ci <run-url>` |
| Unassigned PRs | `/claim-unassigned-prs <repo>` |
| Stale PRs | `/nudge-stale-prs` |
| Node behind LTS | `/upgrade-node-version` |

When several repos share a condition, also suggest the fleet variant (`/sweep-all-dependency-prs`, `/sweep-all-dependabot-alerts`, `/upgrade-node-across-org`).

## 3. Publish the dashboard

**Artifact-capable agent with a shared database** (e.g. Claude Code's artifact tool):

- If the agent offers a **Dashboard** artifact type, create the dashboard from that type and feed it the collected rows. Otherwise write the page yourself and give it the **database capability**, so the page reads its rows at runtime instead of baking data into the HTML.
- Store data as documents in collections (schema in [references/data-model.md](references/data-model.md)): `repos` (one document per repo, keyed by `owner__repo`), `runs` (one document per refresh: timestamp, counts, duration), and `meta` (owned orgs, filters).
- Write rows with a **batch** write, never one call per repo.
- The page shows: KPI tiles (repos, 🔴/🟡/🟢 counts, open alerts by severity, open bot PRs), a filterable and sortable table (status, repo, signals, next action with a copy-to-clipboard command), a per-org breakdown, a trend sparkline from `runs`, and "last refreshed" with its age. Works in light and dark themes and at phone width.
- Links go to GitHub (PR list, alert, failing run) so every cell is one click from the fix.

**Otherwise:** write a self-contained HTML page with the data inlined, outside any repo's tracked tree, open it locally, and print the 🔴 rows in the chat.

Record the dashboard URL in the report so `refresh` can find it; if unsure on a later run, list the user's artifacts and match by title (`Fleet Health`).

## 4. Refresh

`refresh` re-runs steps 1–2 and updates the existing database rows (upsert changed repos, delete rows for repos that left the union, append one `runs` document). Do not republish the page unless its code changed. Report what changed since the previous run: newly red, newly green, alerts opened and closed.

## 5. Report

```markdown
# Fleet health — <date>

**Repos:** N (personal P · owned orgs O · other controlled C) · 🔴 R · 🟡 Y · 🟢 G
**Dashboard:** <url>
**Since last refresh:** +a red, −b red, +c alerts, −d alerts

## Needs you (🔴)
| Repo | Why | Next action |
| --- | --- | --- |
| owner/repo | 2 critical alerts, main red | `/sweep-all-dependabot-alerts critical` · `/fix-failing-ci <run>` |

## Probe gaps
- n/a cells by reason (Dependabot disabled, 403, rate limited)
```

Offer to run the suggested skills, but do not run any of them without the user's go-ahead.

## 6. Schedule (optional)

Only when the user asks. Prefer, in order:

1. **Cloud routine** (Claude Code `/schedule`): a weekday-morning routine whose prompt is `/fleet-health-dashboard refresh`. It needs GitHub access in that environment — confirm the routine can authenticate `gh` before relying on it.
2. **Local loop** (`/loop`) for a session that stays open.
3. **cron / launchd** calling the agent's CLI non-interactively.

State the schedule, the time zone, and how to stop it.

## Guardrails

- Read-only on GitHub: no merges, approvals, comments, or reviewer requests from this skill.
- The dashboard is private by default; private repo names stay out of anything shared unless the user shares it.
- Never store tokens or alert details beyond what GitHub already shows the same user.
