# Trainiac Feature Request Dashboard

**Live URL:** https://lilit-wellhub.github.io/trainiac-feature-requests/
**Repo:** https://github.com/lilit-wellhub/trainiac-feature-requests
**Owner:** Lilit Arutyunyan — PM, New Ventures / Wellhub
**Last updated:** 22/06/2026

---

## What it is

A single-page internal dashboard for tracking feature requests across the Trainiac product. It consolidates signals from multiple sources (CX tickets, App Store reviews, Service Desk, CS), counts them against a threshold, and surfaces which requests need a PRD.

It's a static HTML file with no backend — all data lives in the browser's `localStorage`. It can be opened locally or accessed at the GitHub Pages URL above.

---

## How it works

### Data model

Each feature request is an object with these fields:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Sequential ID — `FR-001`, `FR-002`, etc. |
| `name` | string | Short description of the feature |
| `count` | number | Total number of signals (tickets, reviews, mentions) logged |
| `firstLogged` | string (DD/MM/YYYY) | Date of the first signal |
| `lastLogged` | string (DD/MM/YYYY) | Date of the most recent signal |
| `sources` | string[] | Where signals came from: `CX`, `App Store`, `Service Desk`, `CS`, `In-app` |
| `persona` | string | Who raised it: `User` (client-facing app), `Trainer` (T4T platform), or `Both` |
| `os` | string | Platform: `iOS`, `Android`, or `Both` — blank for Trainer (T4T is web) |
| `status` | string | Pipeline stage (see below) |
| `renewalSignal` | boolean | Whether CS flagged this as a renewal risk (lowers threshold to 5+) |
| `notes` | string | Context, member quotes, ticket references |
| `jiraLink` | string | URL to the Jira epic or ticket |
| `prdLink` | string | URL to the Google Doc PRD |

### Pipeline stages (status)

| Status | Meaning |
|--------|---------|
| 🟡 Logged | Signal received, not yet reviewed |
| 🔵 Under Review | PM is assessing scope and priority |
| 🟠 PRD Ready | At threshold — PRD needed within 5 business days |
| 🟢 In Jira | Jira epic/ticket created, in backlog or in progress |
| ✅ Shipped | Feature has been released |
| ❌ Declined | Intentionally not building; documented rationale |

### Threshold logic

A request is flagged as **at threshold** (shown with a red `THRESHOLD` badge and counted in the stats bar) when:
- `count >= 10` for any source, OR
- `count >= 5` AND `renewalSignal = true` (CS-flagged renewal risk)

Requests at threshold and not yet at `PRD Ready` or beyond trigger the yellow warning banner at the top of the dashboard.

---

## Filters and views

### Filter pills
- **Status** — filter by pipeline stage
- **Source** — filter by where signals came from (CX, App Store, Service Desk, CS)
- **Persona** — filter by who it affects: Trainer (T4T platform), User (client-facing app), or Both

### Sort options
Count (high/low), last logged date (newest/oldest), status order

### Search
Full-text search across feature name, notes, and ID

### Views
- **Table** — sortable list with all fields visible
- **Pipeline** — kanban-style columns by status (excludes Declined)

---

## Stats bar

| Stat | Definition |
|------|-----------|
| Total requests | All FRs in the registry |
| At threshold · needs PRD | FRs at threshold AND not yet in PRD/Jira/Shipped stage |
| In flight | FRs in Under Review, PRD Ready, or In Jira |
| Shipped | FRs with ✅ Shipped status |
| Total member signals | Sum of `count` across all FRs |

---

## Adding and editing requests

Click **+ Add request** (top right) to log a new FR. Fill in:
- Feature name (required)
- Count, status, first/last logged dates
- Persona (User / Trainer / Both) + OS if applicable
- Sources (checkboxes)
- CS renewal signal flag
- Notes and member quotes
- Jira and PRD links

Click any row to open the edit modal. From there you can also delete the request.

---

## How new requests are sourced

The Jira leg of intake is **fully automated** via a scheduled Claude task that runs every Monday at 09:00. It queries Jira, updates counts and dates, adds newly clustered FRs, and pushes the updated dashboard to GitHub — no manual steps required. App Store and CS signals are still logged manually via the **+ Add request** button.

### Signal sources

| Source | What it is | Automated? |
|--------|-----------|------------|
| **CX** | Customer support tickets in Jira (projects: `TBT`, `MAIN`, `NVT`) | ✅ Weekly — scheduled task |
| **Service Desk** | Trainer tickets in Jira NVT | ✅ Weekly — scheduled task |
| **App Store** | iOS App Store reviews (app ID: 1244920288) | ✅ Weekly — Apple RSS feed, up to 10 pages |
| **Google Play** | Android reviews (package: `fit.trainiac.android.client`) | ✅ Weekly — Play Store page scrape |
| **CS** | Renewal signals from CS team (Slack / Salesforce) | Manual — log via + Add request when flagged |

### Automated Jira sync (every Monday 09:00)

The scheduled task (`trainiac-fr-sync`) runs four JQL queries:

**Query A — labeled FRs in NVT**
```
project = NVT AND labels in ("trainiac-feature-request","trainiac-feature-request-wontfix") ORDER BY created ASC
```

**Query B — Won't Fix / Not a Bug resolutions in NVT**
```
project = NVT AND resolution in ("Won't Fix","Not a Bug","Works as Designed","By Design") AND labels not in ("trainiac-feature-request","trainiac-feature-request-wontfix") ORDER BY created ASC
```

**Query C — Trainiac CX tickets across TBT/MAIN**
```
project in (TBT,MAIN) AND text ~ "trainiac" ORDER BY created ASC
```

**Query D — keyword scan for untagged FRs**
```
project = NVT AND (summary ~ "wish" OR summary ~ "would be great" OR summary ~ "feature request" OR summary ~ "please add" OR summary ~ "can you add") ORDER BY created ASC
```

Each ticket is matched to an existing FR by keyword similarity. Matched tickets increment `count` and update `lastLogged`. Unmatched tickets with 2+ signals clustering around the same theme are added as new `🟡 Logged` entries. Results are pushed to GitHub and the live dashboard updates within ~60 seconds.

A sync log is appended to `sync-log.md` in this folder after each run.

### Manual intake (CS only)

CS renewal signals are the one source that can't be scraped — they come from conversation context in Slack or Salesforce. When the CS team flags a renewal risk, log it via **+ Add request** and tick the **CS renewal signal** checkbox (lowers the threshold to 5+).
3. Increment `count` and update `lastLogged` for matched FRs
4. Surface unmatched tickets as draft entries for PM review
5. Flag any FRs that crossed the threshold since the last sync

---

## Persona / platform classification

| Persona | Platform | OS |
|---------|----------|----|
| **User** | Client-facing Trainiac app | iOS, Android, or Both |
| **Trainer** | T4T (Trainer for Trainer) web platform | — (web-based, no OS) |
| **Both** | Affects both platforms | iOS, Android, or Both |

---

## Data persistence

Data is stored in `localStorage` under the key `trainiac-fr`. The seed data (21 pre-loaded FRs) is re-applied automatically if:
- No data exists in localStorage, OR
- Fewer entries than the seed data, OR
- Data is missing the `persona` field (schema migration)

This means any additions or edits made in the browser persist across sessions on the same device but are **not synced** across devices or shared with teammates. For shared state, the next step is a backend (see backlog below).

---

## Deployment

The dashboard is a single `index.html` file — no build step, no dependencies, no server required.

**GitHub Pages auto-deploy:** Any push to the `main` branch of `lilit-wellhub/trainiac-feature-requests` triggers a GitHub Actions build and deploys to the live URL within ~60 seconds.

To update the dashboard:
1. Edit `index.html` locally
2. Copy to `/tmp/fr-deploy/`
3. Commit and push — GitHub Pages handles the rest

---

## Backlog / known limitations

| Item | Notes |
|------|-------|
| No shared state | localStorage is per-device; changes aren't visible to teammates |
| Jira sync is a stub | The "Sync Jira" button simulates a sync but doesn't call the API; real integration requires Atlassian MCP or a server-side token |
| No authentication | The dashboard is public (anyone with the URL can view it) |
| Manual count updates | Counts need to be updated manually as new tickets come in |
| Backend + auth | Next milestone: move data to a Supabase table + add Wellhub SSO |

---

## Related files

| File | Description |
|------|-------------|
| `~/Documents/my-os/projects/trainiac/README.md` | Main Trainiac project README |
| `~/Downloads/course-materials/your-work/comms/trainiac-feature-request-registry.md` | Source-of-truth Markdown registry (used before the dashboard existed) |
| `~/Downloads/course-materials/your-work/comms/jira-historical-export-runner.md` | Instructions for running a full Jira backfill into the registry |
