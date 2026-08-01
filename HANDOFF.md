# HR AI Adoption Intelligence Tracker — Agent Handoff

**Last updated:** 2026-07-21
**Owner:** Andrew Helms (andrewmhelms@icloud.com)
**Live site:** https://ai-labsonlabs.github.io/hr-ai-tracker-deploy/
**Repo:** https://github.com/AI-labsonlabs/hr-ai-tracker-deploy
**Prior agent:** Perplexity Computer (Claude Sonnet 4.6)

This document is the operational reference for whichever AI agent takes over ongoing maintenance and development of the tracker. It describes the **current production system as of 2026-07-21**, not the historical path we took to get here. Anything historical, deprecated, or experimental is explicitly labeled.

---

## 1. Product Purpose

### What it is
The HR AI Adoption Intelligence Tracker is a research-driven dashboard that measures HR/People AI adoption across ~428 large enterprises. Each company has a **maturity score** (0-100), a **momentum score** (-10 to +10), an **adoption pattern** label, and per-domain scores across seven HR AI domains. Scores are computed from ~3,800 individual signals (deployment stories, pilots, announcements, hires, upskilling programs, job postings) collected from primary sources.

### Intended users
Product currently serves one primary persona; a homepage redesign for a broader audience is under consideration but not yet built.
- **Primary today:** the site owner, using it as an internal research tool and public artifact.
- **Aspirational (in discussion):** HR AI Program Leads / HRIS Directors evaluating vendors; HR Tech analysts (Josh Bersin readers, HR Executive readers); CHRO/VP People at large enterprises benchmarking peers; HR AI vendors doing competitive intel.

### Primary use cases
- **Peer benchmarking** — "how does my company's HR AI maturity compare to peers in my industry?"
- **Vendor customer discovery** — "which enterprises are actually deploying vendor X?"
- **Trend spotting** — "which HR AI domain has the most momentum this quarter?"
- **Signal source review** — every score traces back to primary-source URLs

### Dashboard areas (`index.html` tabs)
The sidebar navigation exposes seven tabs, each backed by a specific slice of `data.js`:

| Tab (data-tab) | Purpose | Backing data key |
|----------------|---------|------------------|
| `overview` | Landing summary — total companies/signals, industry averages, finserv focus, top movers | `summary`, `top_movers`, `industries`, `finserv_subindustry` |
| `signals` | Chronological signal feed with company, domain, type, source | `company_details[].top_signals` aggregated |
| `tech` | HR AI vendor landscape — 32 tracked technologies with deployment counts | `tech_landscape` |
| `peers` | Peer comparison for a selected company | `company_details` |
| `heatmap` | 7-domain heatmap across 428 companies | `heatmap` |
| `methodology` | Scoring methodology explanation | Static HTML |
| `admin` | Password-gated admin actions (approve/deny requests, close flags) | Calls Cloudflare Worker admin endpoints |

### Key term glossary
- **Company universe** — the ~428 enterprises the tracker follows. Stored in `tracker/company_universe.json` (in the research workspace, **not in this repo**).
- **Signal** — one primary-source-verified data point about an HR AI adoption event at a tracked company. Each signal has a `signal_type`, `HR_domain`, `date_event`, `date_found`, `source_type`, `source_urls`, `summary`, and `company_name`.
- **Admin signals** — the curated, human-approved corpus in `admin_signals.json`. This is the authoritative signal file that ships in the repo.
- **Research corpus (`all_signals.json`)** — the larger, automated research signal pool (~3,656 signals) that the scoring pipeline also consumes. Not in the deploy repo; lives in the research workspace at `tracker/research/all_signals.json`.
- **Maturity score** — blended 0-100 score per company. Formula in §4.
- **Momentum score** — -10 to +10 score reflecting recent signal velocity + acceleration. Formula in §4.
- **PR ratio** — deployment/announcement balance. Labels: "Action-Oriented", "Balanced", "PR-Leaning", "PR-Heavy".
- **Adoption pattern** — categorical label like "Tool-led", "Governance-led", etc., derived from which signal types dominate.
- **Coverage** — number of the 7 domains scoring above 5.0 (the `COVERAGE_THRESHOLD`).

### Signal type controlled vocabulary (authoritative — from `consolidate_scores.py`)
| Code | Label | Base weight |
|------|-------|-------------|
| `A` | Deployment (production use with named use case, ideally quantified outcomes) | 8 |
| `B` | Pilot (POC, limited rollout, not yet production) | 4 |
| `I` | Announcement (partnership, planned rollout, product launch — no confirmed production) | 3 |
| `D` | Leadership Hire (CHRO, HR-AI VP, etc.) | 2.5 |
| `F` | Upskilling (workforce AI training programs, org-wide AI enablement) | 3 |
| `J` | AI Talent Hiring (job posting for HR-AI PM, HRIS AI architect, etc. — investment intent) | 1.5 |

### HR domain controlled vocabulary (7 canonical domains — from `ALL_DOMAINS` in the code)
- `Recruiting` — sourcing, screening, interviewing, ATS, candidate matching
- `Talent Development` — internal mobility, career pathing, succession, talent marketplaces
- `HR Service Delivery` — HR chatbots, case management, employee self-service, onboarding automation
- `Comp/Benefits` — compensation planning, pay equity, benefits administration
- `Governance` — AI policy, HR-AI ethics, bias auditing, compliance
- `AI Hiring` — AI-first interviewing platforms (HireVue, MyInterview, Paradox full solution)
- `People Analytics` — workforce data, HR dashboards, predictive analytics, sentiment analysis

**Known taxonomy drift (see §7):** the admin_signals corpus contains three additional non-canonical domain values used in ad-hoc signals — `Learning & Development` (3 signals), `Employee Engagement` (1), `HR Technology` (1). These do not contribute to canonical domain scores. Should be reconciled in a future cleanup pass.

### Source type vocabulary + multipliers (`consolidate_scores.py:SOURCE_MULTIPLIERS`)
```
vendor case study        1.5    (highest — primary confirmed deployment)
SEC filing               1.5
press release            1.3
company press release    1.3
product release notes    1.3
earnings call            1.2
conference presentation  1.2
conference               1.2
news article             1.1
industry report          1.1
blog post                1.0
job post                 0.8
social media             0.7
edited                   1.0    (human-edited signal)
```

---

## 2. Current Architecture

### Deploy repository layout (this repo — `AI-labsonlabs/hr-ai-tracker-deploy`)

| File | Purpose | Auto-updated? |
|------|---------|---------------|
| `index.html` | Single-page dashboard app (~460 KB, all JS/CSS inline in `<script>`/`<style>` blocks or via `base.css`/`style.css`). Loads `data.js` on page load. | No — hand-edited only |
| `data.js` | The full dashboard data payload. Format: `const DASHBOARD_DATA = {...large JSON...};` | **Yes — every cron run** |
| `admin_signals.json` | Full signal corpus in JSON (same content used to build `data.js`). Shipped so the admin tab can display raw signals. | **Yes — every cron run** |
| `base.css`, `style.css` | Base + main stylesheets | No |
| `andrew-helms.jpg` | Owner headshot for methodology tab | No |
| `README.md` | 83-byte placeholder | No |
| `HANDOFF.md` | This document | No |

### `data.js` payload structure (top-level keys)
- `generated_at` (ISO timestamp)
- `attributes_last_updated` (formatted string, e.g. "Jul 21, 2026")
- `summary` — dashboard header stats (total_companies, total_signals, avg_maturity_finserv, avg_maturity_benchmark, finserv_companies)
- `industries` — 12 industry buckets with rollup stats
- `top_movers` — 428 companies ranked by momentum
- `governance_leaders` — 210 companies ranked by governance-domain score
- `heatmap` — 428 keys, each mapping to 7 domain scores
- `finserv_subindustry` — 5 finserv sub-industry buckets
- `company_details` — 428 keys, each is the full per-company detail record (see §5)
- `signal_type_breakdown` — 6-item list, one per signal type
- `tech_landscape` — 32-vendor landscape

### Production data flow

```
                            [ RESEARCH WORKSPACE ]
                            /home/user/workspace/tracker/
                            (Perplexity sandbox — NOT in git)

  ┌────────────────────┐    ┌──────────────────────────┐
  │ company_universe   │    │ research/                │
  │ .json  (428 cos)   │    │   admin_signals.json     │
  │                    │    │   all_signals.json       │
  └────────┬───────────┘    └────────────┬─────────────┘
           │                             │
           └─────────────┬───────────────┘
                         ▼
                ┌────────────────────┐
                │ consolidate_scores │ ← scoring engine
                │        .py         │   (maturity, momentum, domain scores)
                └────────┬───────────┘
                         ▼
                consolidated_scores.json
                         │
                         ▼
                ┌────────────────────┐
                │ export_dashboard   │ ← reshapes to dashboard format
                │      _data.py      │
                └────────┬───────────┘
                         ▼
                     data.js  ────────┐
                                      │
                                      ▼
                    [ COPY TO DEPLOY REPO ]
                    /home/user/workspace/hr-ai-tracker-deploy/
                    admin_signals.json ← also copied
                                      │
                                      ▼
                              git add + commit + push
                                      │
                                      ▼
                          GitHub Pages (automatic)
                                      │
                                      ▼
              https://ai-labsonlabs.github.io/hr-ai-tracker-deploy/
```

Full pipeline command (used by the daily/weekly crons):
```bash
cd /home/user/workspace/tracker
python3 consolidate_scores.py
python3 export_dashboard_data.py
cp research/admin_signals.json /home/user/workspace/hr-ai-tracker-deploy/admin_signals.json
cd /home/user/workspace/hr-ai-tracker-deploy
git add admin_signals.json data.js
git commit -m "Update dashboard data — YYYY-MM-DD"
git push
```

### External services

| Service | Purpose | Where credentials live |
|---------|---------|------------------------|
| **GitHub** | Repo hosting + GitHub Pages | Agent uses `api_credentials=["github"]` when calling git. Also used for GitHub Issues (request queue). |
| **GitHub Pages** | Static site hosting for the dashboard | No credentials — enabled from repo settings, `main` branch |
| **Cloudflare Workers** (`hr-tracker-proxy`) | Public request-submission proxy + admin UI. Source: `/home/user/workspace/hr-tracker-worker/` (NOT in this repo — separate deploy). URL: `https://hr-tracker-proxy.hr-ai.workers.dev` | `GITHUB_TOKEN` — fine-grained PAT with Issues:read/write, set via `wrangler secret put`. Also a KV namespace `RATE_LIMIT`. |
| **Cloudflare KV** | Rate-limit backing store for the Worker | Namespace binding `RATE_LIMIT`, id `96b88b26fc614b31a338cbf211a1e1d9` in `wrangler.toml` |
| **Perplexity `finance` connector** | Daily market-cap refresh (`finance_quotes` tool) | Used by `_cron_refresh_market_caps.py` via `api_credentials=["external-tools"]` |
| **Perplexity `pplx_sdk.search`** | Web search for signal discovery | Called from cron subagents |

### Public request submission flow
Public users can request new companies/technologies be added:
1. Site JS `POST`s to Cloudflare Worker at `https://hr-tracker-proxy.hr-ai.workers.dev/company-request` (or `/tech-request`, `/flag`)
2. Worker validates payload, rate-limits (5/hr per IP for requests, 10/hr for flags)
3. Worker uses `GITHUB_TOKEN` (server-side) to create a labeled GitHub Issue with `company-request` or `tech-request` label
4. Owner reviews via admin UI at `https://hr-tracker-proxy.hr-ai.workers.dev/admin/requests` (password-gated — password stored in Worker code; **rotate on transition, see §9**)
5. Approved requests are added to `company_universe.json` and the pipeline re-runs
6. A separate scheduled task (`f514e4eb`, see §3) emails the owner weekly if new requests arrived

### Worker endpoints
Public (CORS-locked to `https://ai-labsonlabs.github.io`, rate-limited):
- `POST /company-request` — `{name, website_or_ticker?, page_query?}` → creates issue with label `company-request`
- `POST /tech-request` — `{name, vendor?, category?, website?, page_query?}` → creates issue with label `tech-request`
- `POST /flag` — `{signal_id?, company_name?, reason, details?}` → creates issue with label `flag`
- `POST /analytics` — batched event pings, appended as comments on the analytics issue (issue #10)

Admin (password-gated in `worker.js`):
- `GET /admin/requests`, `POST /admin/approve`, `POST /admin/deny`
- `GET /admin/tech-requests`, `POST /admin/tech-approve`, `POST /admin/tech-deny`
- `GET /admin/flags`, `POST /admin/close-flag`
- `GET /admin/analytics`, `POST /admin/verify`

---

## 3. All Recurring Activities

There are **3 active scheduled tasks** at time of writing, all managed as Perplexity scheduled tasks (crons) tied to this workspace. **These tasks are Perplexity-platform artifacts and must be recreated in the new agent's task scheduling system — they cannot be exported as code.** For each below the full task instruction text is reproduced verbatim; the current production text is also stored inside each cron and can be exported with:

```bash
pplx-tool schedule_cron <<'JSON'
{"action":"list"}
JSON
```

(Perplexity CLI. Note: the `list` display truncates task text at ~10K chars but the stored version can be longer. Current stored lengths: daily=~13.9K, weekly=~11.6K, request-notify=~4.4K chars.)

### 3.1 Daily HR AI Signal Scan (Light, rotating)

- **Cron id (Perplexity):** `1c8bfaa8`
- **Currently enabled:** yes
- **Cron expression:** `0 14 * * *` (UTC) → **7:00 AM PDT** / 6:00 AM PST daily
- **Background:** yes (runs without user interaction)
- **Purpose:** Refresh public-company market caps daily and surface a small number of new HR AI adoption signals via rotating source-mode + rotating company sweep, so we don't repeatedly re-crawl the same sources every day.

**Rotation state file:** `/home/user/workspace/cron_tracking/1c8bfaa8/rotation_state.json`
```json
{
  "company_cursor": 208,           // next index into alphabetical universe list
  "vendor_cursor": 2,              // next vendor batch index (0..4)
  "general_queries_last_run": "2026-07-20"
}
```

**Files read:**
- `/home/user/workspace/tracker/company_universe.json`
- `/home/user/workspace/tracker/research/admin_signals.json`
- `/home/user/workspace/cron_tracking/1c8bfaa8/rotation_state.json`

**Files modified:**
- `/home/user/workspace/tracker/company_universe.json` (market caps daily; sometimes aliases)
- `/home/user/workspace/tracker/research/admin_signals.json` (append new signals)
- `/home/user/workspace/cron_tracking/1c8bfaa8/rotation_state.json`
- `/home/user/workspace/cron_tracking/1c8bfaa8/run_log_YYYY-MM-DD.{md,json}`
- `/home/user/workspace/cron_tracking/1c8bfaa8/market_cap_refresh_YYYY-MM-DD.json`
- Deploy repo: `data.js`, `admin_signals.json` (via pipeline + git push)

**Commit message convention:** `Update dashboard data — YYYY-MM-DD` (market-cap-only days) or `Update dashboard data — YYYY-MM-DD: +<Company> <domain> signal` (signal days).

**What happens with no new signals:** Market cap refresh runs regardless. If no new signals AND caps changed, dashboard is regenerated and pushed anyway (market cap changes count). If nothing changed at all, run ends silently — no notification.

**Failure handling:**
- Market cap refresh has structured failure modes documented in the task text (batch failures, missing tickers, stale tickers). The helper script `_cron_refresh_market_caps.py` handles atomic writes and never overwrites good values with null/zero.
- Consecutive escalations (2+ [BACKGROUND CRON ESCALATION] messages for the same cause) should trigger cron deletion per Perplexity task-scheduling guidance. In practice, all escalations to date have been fixable by clarifying task text.

**Dependencies on other tasks:** None. Runs independently.

**FULL TASK INSTRUCTION TEXT (verbatim, current production version):**

<details>
<summary>Click to expand daily cron task text (~13.9K chars)</summary>

```
Run the daily light HR AI signal scan for the HR AI Adoption Intelligence Tracker — with rotating source modes and rotating company sweeps to surface fresh signals.

## Diagnostic note
A query-rotation strategy was introduced after May 17 2026 because the same 7 static queries had stopped producing new signals (2 added in May vs 13 in April). Log the rotation slot (day-of-week mode + which company batch) at the top of the run log so we can compare velocity before/after.

## Context
- Project: HR AI Adoption Intelligence Tracker (428+ tracked companies, 7 HR AI domains)
- Site: https://ai-labsonlabs.github.io/hr-ai-tracker-deploy/
- GitHub repo: AI-labsonlabs/hr-ai-tracker-deploy
- Pipeline scripts: /home/user/workspace/tracker/ (consolidate_scores.py, export_dashboard_data.py, push_to_github.py)
- Universe: /home/user/workspace/tracker/company_universe.json
- Admin signals: /home/user/workspace/tracker/research/admin_signals.json
- Rotation state: /home/user/workspace/cron_tracking/1c8bfaa8/rotation_state.json (read at start, write at end — see Step 0b)
- GitHub CLI available with api_credentials=["github"]

## Step 0a: Refresh market caps for all public companies (live data)
Refresh `market_cap` daily so dashboard caps don't drift. This runs every day regardless of rotation mode.

**Use the helper script** at `/home/user/workspace/tracker/_cron_refresh_market_caps.py`. It handles ticker collection, batched `finance_quotes` calls, CSV download/parsing, staleness checks, currency filtering, and atomic universe writes. Run it like this with `api_credentials=["external-tools"]`:

    python3 /home/user/workspace/tracker/_cron_refresh_market_caps.py \
      --tracking_dir /home/user/workspace/cron_tracking/1c8bfaa8 \
      --today $(date -u +%Y-%m-%d) \
      --batch_size 50

The script writes `market_cap_refresh_YYYY-MM-DD.json` into the tracking dir with these counters: `tickers_requested`, `caps_updated`, `stale_skipped`, `no_data_skipped`, `non_usd_skipped`, `parse_errors`, `batches`, `batches_failed`, `rows_parsed`, `tickers_missing_from_response`.

**Expected healthy values:** On a normal day all ~360 tickers should come back. Expect `caps_updated` ≈ 350, `tickers_missing_from_response` = 0, `batches_failed` = 0. `stale_skipped` is normally <15 — these are delisted/merged companies whose tickers need universe maintenance (not script bugs).

**When to investigate:**
1. `batches_failed > 0` — inspect the `diagnostics` array. Possible phases: `external_tool_nonzero` (tool call failure), `no_csv_file` (response missing csv_files), `csv_download_failed` (signed URL fetch failed), `csv_parse_failed` (header drift), `csv_empty` (CSV had no data rows). Log the phase + batch_head to the run log.
2. `tickers_missing_from_response > 20` — a specific ticker symbol probably needs fixing in `company_universe.json` (foreign listing, name change, acquired/delisted). Cross-check the missing list against `stale_skipped` rows.
3. `stale_skipped` rising over weeks — companies are getting delisted faster than universe maintenance is keeping up.

**How it works (so you know what to fix if the schema drifts):**
- Calls `external-tool call` with `{"source_id":"finance","tool_name":"finance_quotes","arguments":{"ticker_symbols":[...],"fields":["marketCap","currency"]}}` — parameter is `ticker_symbols` (NOT `tickers`).
- The response includes a `csv_files` array. The script downloads `csv_files[0].url` (a signed CloudFront URL) and parses it with the standard `csv` module. **It does NOT parse the markdown table in the `content` field** — that markdown is rendered for human display and routinely truncates on large batches, dropping rows. The CSV is the canonical, complete data source.
- CSV header (stable): `symbol,name,timestamp,fetched_at,market_status,marketCap,currency`
- Skips non-USD rows, rows older than 90 days, and rows with missing/zero marketCap.
- Stores `market_cap` in $M (millions) — always divides raw dollars by 1,000,000.
- Writes universe atomically (.tmp then `os.replace`).
- Never overwrites a previously-good value with 0 or null.

**This step does NOT push to GitHub on its own.** If only market caps changed (no new HR signals), push the universe-derived dashboard at the end of this step:

    cd /home/user/workspace/tracker
    python3 consolidate_scores.py
    python3 export_dashboard_data.py
    python3 push_to_github.py

Use `api_credentials=["github"]` for the push. If HR signals are also added later in the run, Steps 5/6 will handle the push instead — don't double-push.

## Step 0b: Determine today's rotation slot
Compute today's UTC date. Day-of-week (Mon=0..Sun=6) selects the source mode:
- Mon (0) → Vendor customer sweep
- Tue (1) → Earnings & SEC filings
- Wed (2) → Job postings as deployment signals
- Thu (3) → Vendor customer sweep (different vendor subset)
- Fri (4) → Earnings & SEC filings (different angle)
- Sat (5) → Conferences, podcasts, executive interviews
- Sun (6) → HR-specialty publication sweep

Load rotation_state.json if it exists. It contains company_cursor (int, next index into universe alphabetical list), vendor_cursor (0..4), general_queries_last_run (YYYY-MM-DD). If the file does not exist, initialize all counters to 0.

## How to run web searches in this cron
Use the `search_web` tool directly for all queries in Steps 1 and 2 — pass queries in parallel batches of 2-3 per call. **Do NOT attempt to invoke `python -m pplx_sdk.exec` or any `pplx_sdk` CLI module** — that module does not exist in this environment and will fail with `ModuleNotFoundError`. The `pplx_sdk` Python package is importable for library use, but the daily scan doesn't need it — `search_web` is the supported path.

## Step 1: Source-mode queries (run 4–6 queries based on today's mode)

**Vendor customer sweep (Mon/Thu)** — pick 4 vendors from this list based on vendor_cursor, mod 5:
Batch 0: Workday AI, SAP SuccessFactors AI, Oracle HCM AI, ServiceNow HR AI
Batch 1: Eightfold, Phenom, Paradox Olivia, HiredScore
Batch 2: Gloat, Workera, Degreed AI, Cornerstone AI
Batch 3: Visier, Crunchr, One Model, Perceptyx AI
Batch 4: LinkedIn Hiring Assistant, UKG AI, Fetcher, Findem
For each vendor: `"{vendor}" customer case study enterprise 2026`
Increment vendor_cursor (mod 5) at end of run.

**Earnings & SEC filings (Tue/Fri)** — run these queries (rotate emphasis Tue vs Fri):
Tue: `"people analytics" OR "AI HR" earnings call transcript 2026`; `CHRO AI productivity earnings 2026`; `"AI" "HR" site:sec.gov 2026`; `"human capital" AI 10-K OR 10-Q 2026`
Fri: `"workforce AI" earnings call enterprise 2026`; `"AI agents" employees earnings call 2026`; `"AI deployment" CHRO quote 2026`; `"talent acquisition" AI earnings 2026`

**Job postings (Wed)**:
- `"experience with Workday AI" OR "experience with Eightfold" site:linkedin.com/jobs`
- `"experience with Gloat" OR "experience with Phenom" site:linkedin.com/jobs`
- `"prompt engineering" "people analytics" site:greenhouse.io OR site:lever.co`
- `"HR AI product manager" OR "AI HR program manager" 2026`

**Conferences/podcasts/exec interviews (Sat)**:
- `"HR Tech 2026" customer keynote OR case study`
- `"Workday Rising" 2026 customer`
- `"Transform 2026" CHRO OR "chief people officer"`
- `"HR Executive" podcast AI CHRO 2026`
- `"chief people officer" AI interview 2026`

**HR-specialty publications (Sun)**:
- `site:hrexecutive.com AI deployment 2026`
- `site:hrbrew.com AI deployment 2026`
- `site:joshbersin.com AI HR 2026`
- `site:peoplematters.in AI HR 2026`
- `site:diginomica.com HR AI 2026`

## Step 2: Company rotation sweep (every day, in addition to source-mode queries)
Load company_universe.json and produce a sorted list of company names. Starting at company_cursor, pull the next 10 companies (wrap to 0 if past end). For each company run a single query: `"{company}" HR AI OR "people analytics" OR "AI recruiting" 2026`.
Set company_cursor = (company_cursor + 10) % len(universe). With ~410 companies that gives full coverage every ~41 days.

## Step 3: Evaluate candidates against quality gates
For each candidate result mentioning a tracked company:
- Credible primary source (vendor case study, press release, earnings call, news article, blog post, job post, social media, SEC filing, conference, product release notes)
- Named HR-specific use case (not just "using AI" generically; the use case must be tied to a specific HR function)
- Not already in admin_signals.json (check by company_name + any source_url in source_urls)
- Correct classification per the rules below
- **CRITICAL: `company_name` MUST match a key in `company_universe.json` OR an entry in that company's `aliases` array.** If the source uses a different name variant, either pick the universe canonical spelling OR add the variant to the canonical entry's `aliases` array. If the company isn't tracked at all, SKIP the signal and log the candidate + URL in the run log's `notable_candidates_not_added` array. **Never add a company_name that doesn't exist in the universe (and isn't an alias)** — it will be silently orphaned.

## Classification rules
**signal_type** (single letter): A=Deployment (production, named use case, ideally quantified), B=Pilot, I=Announcement (partnership/planned/product launch, no confirmed production), J=Job posting (hiring for HR AI role, investment intent), F=Upskilling (workforce AI training programs, not F=funding). Do NOT invent other letters.

**HR_domain** (exact match required): Recruiting, Talent Development, People Analytics, HR Service Delivery, AI Hiring, Governance, Comp/Benefits. Governance signals MUST be classified as type I.

**Signal object schema** (append to admin_signals.json):
{
  "company_name": "<canonical universe name>",
  "signal_type": "A|B|I|J|F",
  "HR_domain": "<one of the domains>",
  "source_type": "press release|case study|earnings call|news article|blog post|job post|SEC filing|conference|product release notes|social media|vendor case study",
  "date_event": "YYYY-MM-DD",
  "date_found": "YYYY-MM-DD (today UTC)",
  "source_urls": ["https://..."],
  "summary": "<1-3 sentence summary quoting the key metric or use case>"
}

## Step 4: Merge new signals into admin_signals.json (atomic write)
Load admin_signals.json, append qualifying new signals, write atomically (.tmp then os.replace).
Save run log to /home/user/workspace/cron_tracking/1c8bfaa8/run_log_YYYY-MM-DD.json with: run_utc_date, rotation_slot, market_cap_refresh, signals_added, notable_candidates_not_added, company_rotation_sweep.

## Step 5: Rebuild dashboard and push to GitHub
    python3 consolidate_scores.py
    python3 export_dashboard_data.py
    cp research/admin_signals.json /home/user/workspace/hr-ai-tracker-deploy/admin_signals.json
    cd /home/user/workspace/hr-ai-tracker-deploy && git add admin_signals.json data.js && git commit -m "Update dashboard data — YYYY-MM-DD" && git push

Use `api_credentials=["github"]` for the push. If Step 0a already pushed the market-cap-only update earlier in this run, still push here — the signal changes are new. If NO signals AND Step 0a already pushed, skip this step.
```

</details>

### 3.2 Weekly Deep HR AI Signal Scan

- **Cron id:** `64f9ba23`
- **Currently enabled:** yes
- **Cron expression:** `0 15 * * 1` (UTC) → **Mondays 8:00 AM PDT** / 7:00 AM PST
- **Background:** yes
- **Purpose:** Deeper signal discovery pass with an expanded query set, executive quote sweep at top-50 companies, internal AI platform launches, and tiered attribute validation.
- **Recent effectiveness:** 0 net-new signals in 2 consecutive weeks (Jul 13, Jul 20) after strict-filter — the daily rotation is doing most of the real work. **A trim to just Parts 3-6 (executive sweep + internal AI + source-specific + attribute validation) has been discussed but not yet shipped.**

**Files read:** `company_universe.json`, `consolidated_scores.json`, `research/admin_signals.json`
**Files modified:** `research/admin_signals.json`, `company_universe.json` (attribute validation), `cron_tracking/64f9ba23/*.{json,md}`, deploy repo `data.js` + `admin_signals.json`

**Helper scripts referenced:** `/home/user/workspace/tracker/_weekly_strict_filter.py` (161 lines; drops noisy domains, generic titles, substring traps; keeps by URL pattern; dedups vs existing signals). If this script disappears the task text has embedded equivalent logic.

**Commit message convention:** `Weekly deep scan YYYY-MM-DD: +N signals` (or similar).

**FULL TASK INSTRUCTION TEXT — reproduce from `pplx-tool schedule_cron` list output on transfer.** The task is ~10K displayed / ~11.6K stored and includes 6 parts (broad search, vendor customer sweep, executive sweep at top-50 by maturity, internal AI platform sweep, source-specific sweeps for Business Insider / LinkedIn News, tiered attribute validation). Key structural notes for the new agent:

1. Uses `pplx_sdk.search.web_many` for broad search — **must write the raw output to a JSON artifact file BEFORE parsing** (do not regex-parse stdout). Contract: `{results: [{ok, spec, result: [{url, title, domain, snippet}, ...]}], total, saved_to}`. Load with `json.load(...)`, flatten `data['results'][*]['result']` into a hits list.
2. Then run `_weekly_strict_filter.py` on the flattened hits.
3. For Part 3 (executive sweep top-50), rank by `maturity_score` from `consolidated_scores.json` (NOT `company_universe.json` — universe has no scores). Fallback to `annual_revenue` if scores are missing.
4. Part 6 attribute validation is tiered: Tier 1 (top-50) weekly; Tier 2 (other public) in earnings months only; Tier 3 (private) if >56 days stale.

**Failure handling:** This cron has **escalated twice** (2026-07-06, 2026-07-13, 2026-07-20). Per Perplexity task-scheduling guidance, 2+ consecutive escalations for the same cause should trigger deletion. In practice all escalations have been fixable by task-text clarifications. The current pattern is that the subagent completes the search + filter but runs out of budget on manual extraction — an acceptable partial-completion mode that the parent agent finishes.

**Dependencies:** None on other crons.

### 3.3 Notify on new HR tracker requests

- **Cron id:** `f514e4eb`
- **Currently enabled:** yes
- **Cron expression:** `0 15 * * 5` (UTC) → **Fridays 8:00 AM PDT** / 7:00 AM PST
- **Background:** yes
- **Purpose:** Check GitHub for new `company-request` or `tech-request` issues submitted via the Cloudflare Worker, and email the owner a weekly digest if any arrived.

**Files read:** `/home/user/workspace/cron_tracking/request_notify/state.json`, GitHub Issues via `gh` CLI
**Files modified:** `/home/user/workspace/cron_tracking/request_notify/state.json`, `/home/user/workspace/cron_tracking/request_notify/runs.jsonl`

**State file schema:**
```json
{
  "last_checked_iso": "YYYY-MM-DDTHH:MM:SSZ",
  "seen_issue_numbers": [10, 42, 43, ...]     // bounded at 500
}
```

**What happens with zero new requests:** exits silently — no email, no notification, only state-file update.

**Email:** Uses `send_notification` with `channels=["email","in_app"]`, template `generic`, subject "N new HR tracker request(s) this week — review and approve".

**Dependencies:** Depends on the Cloudflare Worker being live and correctly labeling issues as `company-request` / `tech-request`. If the Worker goes down, requests stop arriving, but this cron doesn't detect that — it just reports zero requests.

**FULL TASK INSTRUCTION TEXT (verbatim, ~4.4K chars):**

<details>
<summary>Click to expand request-notify cron task text</summary>

```
Check for new company/tech requests submitted to the HR AI Adoption Intelligence Tracker GitHub repo and email the user a weekly digest if any new ones arrived since the last check.

## Context
- Repo: AI-labsonlabs/hr-ai-tracker-deploy
- Public worker submits requests as GitHub issues with labels `company-request` (for new companies) or `tech-request` (for new HR AI vendors/tools)
- User email: andrewmhelms@icloud.com
- State file: /home/user/workspace/cron_tracking/request_notify/state.json — tracks which issue numbers have already been notified about so we never email the same request twice
- Schedule: Fridays 8am Pacific (15:00 UTC)

## Step 1: Load state
Read state.json. If missing, treat seen_issue_numbers as [] and last_checked_iso as "".

## Step 2: Fetch open requests from GitHub
    gh issue list --label company-request --state open --json number,title,body,createdAt,author,url --repo AI-labsonlabs/hr-ai-tracker-deploy --limit 50
    gh issue list --label tech-request --state open --json number,title,body,createdAt,author,url --repo AI-labsonlabs/hr-ai-tracker-deploy --limit 50

    (Note: the `gh` CLI removed the `user` and `html_url` JSON fields in favor of `author` and `url`. Use `author.login` for the submitter's username and `url` for the issue's HTML link.)
Merge results, tagging each with request_type = 'company' or 'tech'.

## Step 3: Filter to NEW requests
Keep only issues whose number is NOT in seen_issue_numbers. Parse JSON request-data block from issue body (delimited by `<!-- request-data` and `-->`).

## Step 4: Decide whether to notify
Zero new issues → exit silently, no email, no send_notification. Update state file with last_checked_iso = now and end run.
≥1 new issue → build email and send (Step 5).

## Step 5: Send email
send_notification with channels=["email","in_app"], title="{N} new HR tracker request{s} this week", schedule_description="Fridays · 8am PT", body=markdown summary (one section per request with type, name, optional fields, GitHub link, submitter, timestamp), email_args={template: "generic", subject: "{N} new HR tracker request{s} this week — review and approve"}. End body with reminder about admin UI URL and password.

## Step 6: Update state file (atomic)
Write state.json with last_checked_iso = now (UTC ISO 8601), seen_issue_numbers = existing + newly notified (deduped, sorted). Bound at 500 entries.

## Step 7: Save run log
Append one-line JSON to /home/user/workspace/cron_tracking/request_notify/runs.jsonl with timestamp, new_issue_count, new_issue_numbers, total_open_company_requests, total_open_tech_requests, notification_sent. Bound at 200 lines.
```

</details>

---

## 4. Research Methodology

### Signal discovery methodology
Signals are discovered through three complementary streams:

1. **Daily rotating scan** (cron 3.1) — source-mode rotation (7 modes, 1 per day of week) + company sweep (10 cos/day, full universe every 41 days). This is the primary discovery mechanism today.
2. **Weekly deep scan** (cron 3.2) — broader query set + executive quote sweep at top-50 companies + internal AI platform launches + Business Insider/LinkedIn News sweeps.
3. **Public request submissions** — via the Cloudflare Worker, users can request new companies/technologies. Owner reviews weekly.

### Source-quality hierarchy (highest to lowest, from `SOURCE_MULTIPLIERS`)
1. **Vendor case study, SEC filing** (1.5×) — primary confirmed deployment or regulated disclosure
2. **Press release, product release notes** (1.3×)
3. **Earnings call, conference presentation** (1.2×)
4. **News article, industry report** (1.1×)
5. **Blog post** (1.0×)
6. **Job post** (0.8×)
7. **Social media** (0.7×)

### Credibility gates (applied by every discovery run)
- **Primary source required.** Category roundups, aggregator blogs, and generic third-party digests are rejected. Concretely, the strict filter drops these domains unconditionally: `shrm.org`, `joshbersin.com`, `youtube.com`, `medium.com`, `hrtechnologyconference.com`, `appsruntheworld.com`, `finance.yahoo.com`, `techrseries.com`, `hrtechsaas.com`, `hrdconnect.com`, `crowell.com`, `workstep.com`, `tommasomariaricci.com`, `f6s.com`, `deloitte.com`, `cygnet.one`, `ai-scanner.com`, `pdpspectra.com`, `byrxj.com`, `corporateaiconsultants.com`, `leafsrls.com`, `supalabs.co`, `sanalabs.com`.
- **Named HR-specific use case.** Generic "using AI" mentions without a specific HR function (recruiting, HRSD, people analytics, etc.) are rejected. Company-wide productivity rollouts like the Accenture/Microsoft Copilot 743K rollout and Cisco AI agents are recurring **false positives** — company-wide AI enablement without an explicitly HR-branded use case doesn't qualify.
- **Substring traps.** Company names that are common English words (Intel, Target, Meta, Progressive) require additional context validation to avoid substring false positives. Handled by `_weekly_strict_filter.py`.
- **URL-pattern positive signals.** URLs containing `/customers/`, `/case-stud`, `/press-release`, `/earnings`, `/investor`, `/case-study` are prioritized as likely primary sources.

### Duplicate detection
- **Signal-level:** dedup by exact URL match against existing `source_urls` in `admin_signals.json`. This is the primary dedup key.
- **Company-level:** dedup via canonical name + `aliases` array. The `aliases` field on any universe entry is checked case-sensitively before adding a signal. See §5 for the alias mechanism.

### Handling conflicting claims
- Prefer the primary source (vendor case study > news article > blog post).
- If two primary sources conflict on metrics (e.g. "43% cut" vs "50% cut"), use the more recent one and keep both `source_urls`.
- If the company name conflicts across sources (e.g. "Alphabet" vs "Google"), always use the universe-canonical name and add the variant to `aliases`.
- Signals that later prove wrong can be tombstoned in `admin_signals.json` with `"_deleted": true` and a reason field. There are 6 tombstones currently.

### Verification methodology per field
- **Dates (`date_event`, `date_found`):** `date_event` is the actual event date extracted from the source (deployment launch, press release date, earnings call date, publication date if no clearer signal). `date_found` is the UTC date the signal was ingested. Both use `YYYY-MM-DD`.
- **Company names:** MUST match a universe key OR an alias entry. Never invent new names — silent orphaning occurs if a name doesn't match.
- **Tickers:** Verified via the `finance` connector `finance_quotes` tool. Non-USD tickers (European exchanges especially) often need OTC ADR equivalents — for example, `FRVIA` (Euronext Paris) had no USD quote so it was switched to `FURCF` (OTC ADR), and `LIGHT` (Signify, Euronext AMS) was switched to `SFFYF` (OTC ADR).
- **Market cap:** Refreshed daily via `_cron_refresh_market_caps.py`, stored in millions of USD. Non-USD rows skipped. Values older than 90 days are considered stale and skipped. Never overwrites a good value with 0/null.
- **Products / tech:** Cross-referenced against `KNOWN_TECH` dict in `export_dashboard_data.py` (32 vendors currently).
- **Categories:** Controlled vocabulary per `KNOWN_TECH` — e.g. "Core HRIS & Payroll", "Recruiting & ATS".
- **Citations:** Every signal MUST have at least one URL in `source_urls`. The dashboard displays these as clickable citations.

### Scoring and classification calculations

**Domain score (per company × per domain):**
- Raw weight = Σ(base_type_weight × source_multiplier × confirmation_factor × time_decay)
- Governance domain: forced to type-I, multiplier of 6.0× (rare deployments; policy-heavy)
- Half-lives: default 180 days; Governance domain + type-A Deployment signals use 730 days
- Raw weight → 0-100 score via piecewise linear interpolation from `domain_calibration.json`
- Fallback normalization if calibration file missing: `DOMAIN_NORM` dict with per-domain (Kw, Kc) coefficients

**Maturity score:**
```
base_score   = 0.6 * mean(all 7 domain scores) + 0.4 * mean(top 3 domain scores)
coverage_mult = (domains_covered / 7) ^ 0.3    where domains_covered = # domains > 5.0
size_norm     = 1.0 + (4.3 - composite_log) * 0.113   clamped [0.9, 1.201]
                composite_log = 0.5 * log(emp) + 0.5 * log(rev)
complexity    = 0.9 + (regulatory - 1) * 0.05 + (org_complexity - 1) * 0.05  clamped [0.9, 1.3]
final = base_score * coverage_mult * sqrt(size_norm) * sqrt(complexity)
Capped at 100.
```

**Momentum score:**
- Signal windows: 30, 60, 90, 365 days
- `raw_momentum = acceleration * 0.6 + velocity * 0.4`
- `final = 10 * tanh(raw_momentum * 2.5 / 10)`, in range [-10, 10]

**PR ratio:**
- ratio = (deployment signals) / (announcement signals + ε)
- Labels: `Action-Oriented` (≥3.0), `Balanced` (1.5-3.0), `PR-Leaning` (0.5-1.5), `PR-Heavy` (<0.5)

**Adoption pattern:**
- Categorical label derived from which signal types dominate. Examples: "Tool-led", "Governance-led", "Hiring-heavy". See `compute_pattern()` in `consolidate_scores.py`.

### Judgment calls that currently depend on unwritten context (worth codifying)
1. **Governance signals are forced to type I.** This is coded, but the reasoning ("frameworks aren't 'deployments' in the same sense") lives only in the cron task text — should be a code comment.
2. **Company-wide AI Copilot rollouts (Accenture, Cisco pattern) are rejected as HR signals.** This is a repeatedly-rediscovered rule that lives only in agent memory. The rule: if the primary framing is finance/ops/engineering productivity and the HR angle is only "generic workforce upskilling", the signal doesn't qualify. Better codified: require a specific HR function to be named (recruiting, HRSD, people analytics, etc.).
3. **Universe expansion threshold.** Companies with strong primary-source signals but not currently in the universe are queued in `/home/user/workspace/tracker/universe_expansion_queue.md`. The owner has said each new company needs "at least 5 sources" before being added. This is enforced by human review, not code.
4. **When to trim vs delete a failing cron.** After 2 consecutive escalations, the skill guidance says delete. In practice, fixable-by-instruction cases have been fixed rather than deleted. Threshold for actually deleting is fuzzy.

---

## 5. Data Contract

### 5.1 Company universe entry (`tracker/company_universe.json` — dict keyed by canonical company name)

**Schema:**
```json
{
  "company_name": "3M",                       // REQUIRED. String. Matches key.
  "annual_revenue": 33000,                    // REQUIRED. Integer, in $M USD.
  "employee_count": 85000,                    // REQUIRED. Integer.
  "market_cap": 91452,                        // REQUIRED for public. Integer, in $M USD. May be null for private.
  "ticker": "MMM",                            // REQUIRED for public. String. Must resolve on finance connector.
  "exchange": "NYSE",                         // REQUIRED for public. String.
  "public_or_private": "Public",              // REQUIRED. Enum: "Public" | "Private"
  "industry": "Industrials",                  // REQUIRED. Free-string but 12 canonical buckets are known.
  "sub_industry": "Diversified Manufacturing",// REQUIRED. Free-string.
  "geographic_footprint": "Global",           // REQUIRED. Enum: "Global" | "US-focused" | "Regional" (loose)
  "organizational_complexity_proxy": 4,       // REQUIRED. Integer 1-5.
  "regulatory_intensity_score": 2,            // REQUIRED. Integer 1-5.
  "attributes_updated": "2026-07-21",         // REQUIRED. ISO date.
  "aliases": ["3M Company"]                   // OPTIONAL. List of string variants. 17 entries currently use this.
}
```

Canonical industries in use (12): Industrials, Energy, Agriculture, Financial Services, Media/Entertainment, Technology, Healthcare, Consumer Discretionary, Consumer Staples, Real Estate, Utilities, Communication Services.

### 5.2 Signal object (`admin_signals.json` — list of dicts)

**Schema:**
```json
{
  "company_name": "Paychex",                              // REQUIRED. MUST match a universe key or alias.
  "signal_type": "I",                                     // REQUIRED. Enum: A|B|I|D|F|J. See §1 for meanings.
  "HR_domain": "HR Service Delivery",                     // REQUIRED. Enum: 7 canonical values in §1.
  "source_type": "SEC filing",                            // REQUIRED. Enum: see SOURCE_MULTIPLIERS in §1.
  "date_event": "2026-07-17",                             // REQUIRED. ISO date. Actual event date.
  "date_found": "2026-07-21",                             // REQUIRED. ISO date. Ingest date.
  "source_urls": ["https://..."],                         // REQUIRED. List of ≥1 URL.
  "summary": "In its FY2026 10-K, Paychex describes...",  // REQUIRED. 1-3 sentences.
  "_deleted": false,                                      // OPTIONAL. Tombstone flag. If true, signal is excluded from scoring.
  "_delete_reason": "..."                                 // OPTIONAL. Reason for tombstone.
}
```

**Legacy inconsistencies (see §7):**
- 5 signals have `source_type: null` (should be filled).
- 3 signals use non-canonical HR_domain values (`Learning & Development`, `Employee Engagement`, `HR Technology`).
- Older signals sometimes use `source_url` (singular) instead of `source_urls` (list). New code should read from `source_urls` and fall back to `[source_url]` if missing.
- One signal uses `source_type: "vendor blog / press"` (should normalize to `vendor case study` or `blog post`).

### 5.3 Research corpus signal (`tracker/research/all_signals.json` — list of dicts)
Same schema as admin signals, but this file holds the larger, less-curated research pool (3,656 signals). It is also consumed by `consolidate_scores.py` (weighted lower than admin signals via the `edited` source_type when applicable).

### 5.4 Consolidated score record (`tracker/consolidated_scores.json`)

```json
{
  "generated_at": "ISO timestamp",
  "ref_date": "YYYY-MM-DD",
  "total_companies": 428,
  "total_signals": 3801,
  "signal_type_counts": { "A": 1064, "F": 422, ... },
  "companies": {
    "3M": {
      "company": { <full universe entry> },
      "maturity_score": 18.8,
      "maturity_details": { "mean_all_7": 12.3, "mean_top_3": ..., "coverage_mult": 0.72, ... },
      "momentum_score": -1.5,
      "momentum_details": { <window stats> },
      "pr_ratio": { "ratio": 3.0, "pr_pct": 25.0, "deploy_pct": 75.0, "label": "Action-Oriented" },
      "adoption_pattern": "Tool-led",
      "top_signals": [ <top 7 signals by weight> ],
      "total_signals": 7,
      "ai_hiring_intensity": 2,
      "tech_stack": [ <detected vendors> ]
    },
    ...
  }
}
```

### 5.5 Dashboard payload (`data.js`)
Emitted by `export_dashboard_data.py`. Format: `const DASHBOARD_DATA = {json};`. Top-level keys listed in §2.

### 5.6 Relationships

```
company_universe (428) ────< aliases (17 entries)
      │
      │ 1..N
      ▼
admin_signals (198) + all_signals (3656)  ── consolidate_scores.py ──► consolidated_scores
                                                                              │
                                                                              ▼
                                                                    export_dashboard_data.py
                                                                              │
                                                                              ▼
                                                                          data.js
```

Every signal MUST resolve its `company_name` to a universe entry. Orphaned signals (no matching universe entry, no matching alias) are excluded from scoring and logged by `_orphan_signal_check.py`.

### 5.7 Example valid records
- **Valid universe entry (public):** see `3M` above.
- **Valid universe entry with aliases:** Google, aliases: `["Alphabet", "Alphabet Inc", "Alphabet Inc."]`.
- **Valid signal (Deployment A):** see the Paychex example above.
- **Valid signal (Job posting J):**
  ```json
  {
    "company_name": "Wells Fargo",
    "signal_type": "J",
    "HR_domain": "HR Service Delivery",
    "source_type": "job post",
    "date_event": "2026-07-15",
    "date_found": "2026-07-15",
    "source_urls": ["https://jobs.hiringourheroes.org/jobs/517669375-senior-lead-product-manager-hr-ai"],
    "summary": "Wells Fargo hiring Senior Lead Product Manager - HR AI to 'define and execute the HR AI product strategy...' referencing Workday, ServiceNow HRSD, SAP, Oracle."
  }
  ```

---

## 6. Deployment and Operations

### How changes reach GitHub Pages
- The site is a static single-page app served by GitHub Pages from `main` branch, repo root
- Any commit to `main` triggers a Pages rebuild automatically (~30-60 seconds to go live)
- No CI/CD pipeline; git push is the deploy

### Branch and commit workflow
- **Only branch:** `main`. No PR workflow; commits go directly to main.
- **Commit message conventions:**
  - Daily automated: `Update dashboard data — YYYY-MM-DD` (may include `: +<summary>` if signals added)
  - Manual: descriptive, past-tense, no strict template

### Environment variables and credentials

**In the Cloudflare Worker (`hr-tracker-proxy`):**
- `GITHUB_TOKEN` — fine-grained PAT with Issues:read/write on `AI-labsonlabs/hr-ai-tracker-deploy`. Stored via `wrangler secret put GITHUB_TOKEN`. **Rotate on transition.**
- KV namespace `RATE_LIMIT` (id `96b88b26fc614b31a338cbf211a1e1d9`) bound in `wrangler.toml`.
- Admin UI password: stored as the `ADMIN_PASSWORD` Cloudflare Worker secret (managed via `wrangler secret put ADMIN_PASSWORD`). Not stored in source or documentation. Owner holds the current value; ask the owner directly if you need it.

**In the Perplexity agent runtime:**
- `api_credentials=["github"]` — GitHub credential for git push (managed by Perplexity platform, no manual setup)
- `api_credentials=["external-tools"]` — used by `_cron_refresh_market_caps.py` to call `finance_quotes`
- `api_credentials=["pplx-tool:schedule_cron"]` — for managing scheduled tasks
- No `.env` files in this repo

**Not in git:** everything under `/home/user/workspace/tracker/` (universe, signals, scripts) and `/home/user/workspace/hr-tracker-worker/` (worker source). These live in the agent's sandbox workspace. **This is a major transition risk — see §9.**

### Verifying the live deployment
1. Visit https://ai-labsonlabs.github.io/hr-ai-tracker-deploy/
2. Confirm the "Last updated" footer matches today's date
3. Confirm `summary.total_companies` matches the universe size and `summary.total_signals` looks reasonable
4. Open browser dev tools → Network → verify `data.js` loads with a fresh timestamp
5. Load a specific company (e.g. IBM) and confirm signals + maturity render

### Expected update time
- Pipeline: ~10-30 seconds locally
- Git push: <10 seconds
- Pages rebuild: 30-60 seconds
- Total from cron trigger to live: **~2-4 minutes**

### Monitoring / health checks
There is currently **no automated health check** on the live site or the Worker. Failures manifest as:
- **Cron escalation** — Perplexity emits `[BACKGROUND CRON ESCALATION]` messages when subagents fail. These land in the owner's active conversation.
- **Owner spot-check** — owner visits the site periodically and notices staleness or broken tabs.

Adding a Pages health probe is on the improvements list (§7).

### Rollback and recovery
- **Pipeline output corruption:** `git revert <bad_commit>` and push. Since the site is a rebuild-from-source-of-truth (universe + signals), also fix the source data before re-running the pipeline.
- **Signal file corruption:** backups exist at `/home/user/workspace/tracker/research/admin_signals.json.bak_*`. Restore, re-run pipeline.
- **Universe file corruption:** backups exist at `/home/user/workspace/tracker/company_universe.json.bak_orphancleanup` and `.bak_ticker_fix`. Restore, re-run pipeline.
- **Cloudflare Worker outage:** requests stop arriving. Traffic-safe — public form submissions just fail with a CORS/network error. No data loss on the tracker side.
- **KV namespace loss:** rate limits reset to zero, no permanent data lost. Public request submission still works.

---

## 7. Current Status

### What's working
- **Daily rotating scan** producing 3-5 new admin signals per week reliably. Total admin_signals: **198**.
- **Market cap refresh** running clean every morning — 371/371 tickers resolving, 0 missing, 0 batch failures for the last 10 days after the FURCF/SFFYF fix.
- **Dashboard rebuild + push** works end-to-end; live site auto-updates every morning UTC.
- **Alias mechanism** working — the "Alphabet vs Google" flip-flop that used to create orphans is now stable via `aliases` on the Google universe entry.
- **Public request submission** via the Worker is live; the weekly notify cron is functional (no requests this week, correctly exited silently).
- **Signal count trajectory:** ~180 → 198 over the last 3 weeks; healthy growth without noise inflation.

### Known bugs and limitations
1. **Task-text display truncation.** The `pplx-tool schedule_cron list` display truncates task text at ~10K chars. The actual stored text can be longer (daily task is ~13.9K chars, weekly ~11.6K). Verified that the full text runs when the cron fires. **New agent must be aware** — do not overwrite stored task text based on what `list` shows or you will destroy Steps 4+5 of the daily task.
2. **Weekly cron marginal value.** 0 net-new signals in 2 consecutive weeks (Jul 13, Jul 20). Parts 1-2 (broad queries + vendor customer sweep) duplicate the daily rotation. Trimming to Parts 3-6 has been discussed but not shipped.
3. **HR_domain taxonomy drift.** 5 admin signals use non-canonical domain values (`Learning & Development`, `Employee Engagement`, `HR Technology`, `null`). Should be reconciled — either add to canonical vocabulary or remap.
4. **`source_type: null` in 5 admin signals.** Missing field, should be filled.
5. **Universe gap pattern.** Daily crons regularly surface strong signals for companies not in the universe. Currently 7 companies queued in `/home/user/workspace/tracker/universe_expansion_queue.md`: ManpowerGroup, 7-Eleven, PostNL, HR Path Group, Timken, Compass Group, Wood, plus Health Connect America. Owner has agreed to a batch expansion but it hasn't been executed yet.
6. **Admin UI password.** Stored as the `ADMIN_PASSWORD` Cloudflare Worker secret (set via `wrangler secret put ADMIN_PASSWORD`). Rotated 2026-07-31 after a prior value was inadvertently included in an earlier revision of this document — history has been rewritten to purge the leaked value. Rotate again on any future ownership transition.
7. **No automated health check** on the live site or Worker.
8. **Stale tickers accumulating.** 11 tickers now flagged as stale (ATUS, CNSL, CPI, EDR, FL, HES, HYMTF, LGF-A, ORAN, PARA, WBA) — these are delisted/merged/renamed companies that need universe cleanup.
9. **No test suite.** All pipeline scripts are run-and-inspect. Regression testing depends on comparing against `current_data.json` (from Apr 28), which is now itself outdated.
10. **Long-tail signal quality.** The 3,656 `all_signals.json` corpus is much less curated than the 198 admin signals. Owner's convention is that admin_signals is the "trusted" set; all_signals is the "raw" set contributing lower weight to scoring.

### Fragile or manually maintained areas
- **`tracker/research/all_signals.json`** — the 3,656-signal research corpus was assembled via multiple earlier batch research runs; there's no active refresh process. It contributes to scoring but is not part of any current cron.
- **`domain_calibration.json`** — piecewise-linear score interpolation curves per domain. Regenerated only when the scoring formula changes (via `calibration_deep2.py`). No test that scoring output stays stable after edits.
- **`KNOWN_TECH` dict in `export_dashboard_data.py`** — 32 hand-curated vendor entries. New vendors don't auto-appear in the tech landscape tab; they must be added here.
- **Cloudflare Worker admin password** — plaintext in `worker.js`.

### Current backlog (in priority order as of 2026-07-21)
1. **Universe expansion batch** — add 7-8 queued companies (ManpowerGroup, 7-Eleven, PostNL, HR Path Group, Timken, Compass Group, Wood, Health Connect America) with light research (≥5 sources each). This unblocks 6+ pending high-quality signals that keep getting re-flagged by daily crons.
2. **Homepage redesign for HR/HR-IT professionals** — in discussion. Not yet built. Considerations documented in prior conversations: primary persona = HR AI Program Leads, hero widget = "where do I stand" benchmark widget, add a weekly Friday digest email, meet audience on LinkedIn.
3. **Weekly cron trim** — cut Parts 1-2 (broad + vendor sweep) that duplicate daily rotation; keep Parts 3-6 (executive quotes at top-50, internal AI platforms, source-specific sweeps, attribute validation).
4. **Taxonomy cleanup** — reconcile the 5 non-canonical HR_domain signals and fill 5 null source_types.
5. **Stale ticker cleanup** — decide per-company whether to remove from universe (delisted/dissolved) or update to new ticker (merged/renamed) for the 11 stale tickers.

### Recommended next three improvements (with reasoning)
1. **Ship the universe expansion batch** — highest immediate ROI. ~8 companies × ~10min light research each yields 6+ new deployment-Type-A signals waiting to be ingested. This is a clear "unblock the pipeline" win.
2. **Weekly Friday digest email to owner + subscribers** — the homepage discussion identified this as the highest-leverage growth mechanism. The daily cron already produces the data; wiring a Friday digest that summarizes the week's new signals + one chart would (a) create audience retention and (b) validate the audience-building thesis before investing in a homepage redesign. Can be built as a new cron in ~1-2 hours.
3. **Codify the "company-wide Copilot rollout doesn't qualify as HR" rule in code** — this is the single most-often-rediscovered judgment call. Adding an explicit check in the signal-classifier (require named HR function keyword: recruiting/hiring/HRSD/people-analytics/etc.) would prevent recurring false positives (Accenture, Cisco) and free up cron budget.

---

## 8. Decision Log

### Design decisions
| Decision | Chosen | Alternatives rejected | Reasoning |
|----------|--------|----------------------|-----------|
| Data storage in git vs a database | Git (JSON files in repo) | Postgres, Supabase | Auditable diffs, no infra to manage, ships with the site, fits ~200KB scale |
| Static site vs SPA framework | Static HTML + inline JS + `data.js` | React/Next.js | No build step, cacheable, fast, easy to inspect. `data.js` fits comfortably in memory. |
| Public submissions via Worker vs directly to git | Cloudflare Worker as proxy | Direct client-side git commits | Keep `GITHUB_TOKEN` off the client; rate-limit at the edge; centralized audit trail via labeled issues |
| Scoring: piecewise linear calibration vs formula-only | Calibration file with fallback formula | Pure formula | Calibration lets us match reference data (Mean abs diff 0.08 vs `current_data.json`); formula-only was less accurate on tail |
| Governance domain treatment | Force type-I, apply 6.0× multiplier | Treat like other domains | Governance is fundamentally policy-work, not deployment; and Governance signals are rarer so need a boost to be visible |
| Signal dedup key | URL match in `source_urls` | Fuzzy title/summary match | Simpler, deterministic, and empirically catches ~99% of dupes because the same story doesn't get rediscovered from different URLs often |
| Universe name normalization | `aliases` array on canonical entry | Global synonym dict | Keeps the canonical name authoritative; alias lookup is O(entries × avg_aliases) which is fine at 428 companies |
| Weekly deep scan retention | Keep with trim planned | Delete after 2 escalations (per skill guidance) | Fixable-by-instruction escalations are worth fixing; deletion is reserved for genuinely unfixable failures |
| Universe expansion cadence | "Clean up periodically when orphans accumulate" | Auto-add companies on any strong signal | Owner preference — quality bar (≥5 sources) matters more than coverage speed |

### Research decisions
- **≥5 credible sources per new company added to universe** — owner's explicit rule (2026-07-03).
- **Aliases used to prevent name flip-flop** (e.g. Alphabet/Google) — owner-approved (2026-07-03) instead of auto-normalizing in code.
- **`_deleted` tombstones** for retracted signals rather than hard delete — preserves audit trail; scoring pipeline ignores tombstoned rows.
- **Company-wide Copilot rollouts rejected as HR signals** — recurring judgment call, not yet codified. See §7 improvement #3.

### Owner preferences that materially shaped the tracker
- **No daily email digests unless there's a real reason to send.** Weekly cadence for the request notifier.
- **Prefer notifying via email over push.** Set `channels=["email","in_app"]` for owner-facing notifications.
- **Never delete cron just because instructions were wrong — fix the instructions.** Owner prefers debugging over churn.

---

## 9. Transition Checklist

Everything a new agent must recreate or verify to safely take over.

### External accounts and services (must remain available)
- [ ] **GitHub** account with write access to `AI-labsonlabs/hr-ai-tracker-deploy`. Currently the owner authorizes agent access via Perplexity's git-agent-proxy.
- [ ] **Cloudflare** account owning the `hr-tracker-proxy` Worker. Wrangler CLI access needed for redeploys.
- [ ] **Perplexity account** with scheduled-task capability (or equivalent in the new agent).
- [ ] **`finance` connector** (or equivalent market-data source) callable from cron subagents. If unavailable, the daily market-cap refresh will fail and needs a replacement data source (`yfinance` Python library is a reasonable fallback).
- [ ] **Owner email** `andrewmhelms@icloud.com` still receives from the notification system.

### Secrets to rotate on transition
- [ ] `GITHUB_TOKEN` in the Cloudflare Worker — regenerate PAT and re-run `wrangler secret put GITHUB_TOKEN`.
- [ ] Admin UI password: rotate the `ADMIN_PASSWORD` Cloudflare Worker secret (`wrangler secret put ADMIN_PASSWORD`) and confirm with the owner via a secure channel — never in git, chat transcripts, or documentation.
- [ ] Any Perplexity-side API credentials tied to the prior agent's identity.

### Files that must be transferred outside git (research workspace)
These live in `/home/user/workspace/tracker/` in the prior agent's Perplexity sandbox and are NOT in the deploy repo:

**Data files:**
- [ ] `company_universe.json` (~190 KB, 428 entries with aliases)
- [ ] `research/admin_signals.json` (~113 KB, 198 signals)
- [ ] `research/all_signals.json` (~3,656 signals — raw research corpus)
- [ ] `domain_calibration.json` (21 KB — scoring calibration)
- [ ] `optimized_domain_params.json` (0.9 KB — calibration params)
- [ ] `consolidated_scores.json` (3 MB — cached scoring output)
- [ ] `current_data.json` (3.4 MB — reference dataset for regression testing; **from Apr 28, stale but used for accuracy comparison**)
- [ ] `universe_expansion_queue.md` (2.8 KB — current backlog of companies with pending signals)

**Scripts:**
- [ ] `consolidate_scores.py` (30 KB — scoring engine)
- [ ] `export_dashboard_data.py` (19 KB — dashboard payload builder)
- [ ] `push_to_github.py` (3 KB — git push helper)
- [ ] `_cron_refresh_market_caps.py` (9 KB — market cap refresh helper for daily cron)
- [ ] `_weekly_strict_filter.py` (6 KB — noise filter for weekly cron)
- [ ] `_orphan_signal_check.py` (3 KB — validates signals-to-universe integrity)
- [ ] `_apply_orphan_cleanup.py` (4 KB — one-shot orphan cleanup, deprecated after 2026-07-03 run)
- [ ] `calibration_deep.py`, `calibration_deep2.py` (calibration regeneration — historical, rarely run)

**Cron tracking artifacts:**
- [ ] `/home/user/workspace/cron_tracking/1c8bfaa8/` — rotation_state.json, run logs, market cap refreshes
- [ ] `/home/user/workspace/cron_tracking/64f9ba23/` — weekly scan artifacts
- [ ] `/home/user/workspace/cron_tracking/request_notify/` — request notifier state

**Cloudflare Worker source (in a separate directory outside this repo):**
- [ ] `/home/user/workspace/hr-tracker-worker/worker.js` (~1000 lines)
- [ ] `/home/user/workspace/hr-tracker-worker/wrangler.toml`
- [ ] `/home/user/workspace/hr-tracker-worker/README.md`

Recommended: mirror the entire research workspace into a private git repo (e.g. `AI-labsonlabs/hr-ai-tracker-research`) rather than depending on the Perplexity sandbox persistence.

### Scheduled tasks that must be transferred to the new agent
For each task, the new agent must (a) recreate the schedule in its own task scheduler and (b) copy the full task instruction text verbatim from the prior agent's Perplexity task store.

- [ ] **Task 1: Daily HR AI Signal Scan (Light, rotating)** — cron `0 14 * * *` UTC. Full instruction text ~13.9K chars, reproduced in §3.1 above.
- [ ] **Task 2: Weekly Deep HR AI Signal Scan** — cron `0 15 * * 1` UTC. Full instruction text ~11.6K chars, must be exported from prior agent via `pplx-tool schedule_cron` list.
- [ ] **Task 3: Notify on new HR tracker requests** — cron `0 15 * * 5` UTC. Full instruction text ~4.4K chars, reproduced in §3.3 above.

### Step-by-step transition validation test

Once the new agent has been onboarded and everything above is in place, run this test to prove the handoff is complete:

1. **Verify site is live**
   - `curl -sI https://ai-labsonlabs.github.io/hr-ai-tracker-deploy/ | grep -E "HTTP|last-modified"` → expect 200 OK
   - Open site in browser, confirm dashboard loads with all 7 tabs

2. **Verify data payload is current**
   - `curl -s https://ai-labsonlabs.github.io/hr-ai-tracker-deploy/data.js | head -c 500` → confirm `generated_at` is <48 hours old

3. **Verify pipeline runs end-to-end from the new agent's environment**
   ```bash
   cd /path/to/tracker
   python3 consolidate_scores.py    # should produce/update consolidated_scores.json
   python3 export_dashboard_data.py # should update data.js in the deploy repo
   ```
   Confirm no errors, `data.js` file mtime updates, and `total_companies` matches universe size.

4. **Verify orphan check passes**
   ```bash
   python3 _orphan_signal_check.py --out /tmp/orphan_check.json
   ```
   Confirm output shows **0 orphans**. If nonzero, at least one signal has a `company_name` that doesn't match a universe key or alias — resolve before proceeding.

5. **Verify market cap refresh works**
   ```bash
   python3 _cron_refresh_market_caps.py --tracking_dir /tmp/test_refresh --today $(date -u +%Y-%m-%d) --batch_size 50
   ```
   Confirm the output JSON shows `caps_updated ≥ 350`, `tickers_missing_from_response = 0`, `batches_failed = 0`.

6. **Verify git push works**
   Make a trivial change (e.g. update `HANDOFF.md` timestamp) → commit → push → confirm GitHub Pages redeploys within 60 seconds.

7. **Verify Cloudflare Worker is live**
   ```bash
   curl -X OPTIONS https://hr-tracker-proxy.hr-ai.workers.dev/company-request \
     -H 'Origin: https://ai-labsonlabs.github.io' \
     -H 'Access-Control-Request-Method: POST'
   ```
   Expect a 204 with CORS headers.

8. **Verify admin UI is accessible**
   Load `https://hr-tracker-proxy.hr-ai.workers.dev/admin/requests` in a browser, enter the (rotated) admin password, confirm any open request issues render.

9. **Verify each scheduled task fires**
   Manually trigger each task once (or wait one natural firing cycle) and confirm:
   - Daily task produces a run log and (if applicable) a git commit
   - Weekly task produces a run log
   - Request-notify task updates state.json

10. **Verify owner reachability**
    Send one test in-app + email notification via `send_notification` to `andrewmhelms@icloud.com` and confirm receipt.

Handoff is complete when all 10 steps pass and the owner has confirmed continued visibility into the daily commit stream.

---

## Appendix — Repository file inventory (2026-07-21)

```
hr-ai-tracker-deploy/                              (this repo, GitHub Pages source)
├── HANDOFF.md              1                     ← this file
├── README.md               83 B                  (placeholder)
├── admin_signals.json      113 KB, 198 signals   (auto-updated by cron)
├── andrew-helms.jpg        45 KB                 (owner headshot)
├── base.css                1.9 KB
├── data.js                 3.5 MB                (auto-updated by cron)
├── index.html              460 KB                (main dashboard SPA)
└── style.css               7 KB

  Research workspace (NOT in this repo, in Perplexity sandbox):

/home/user/workspace/tracker/
├── company_universe.json                194 KB, 428 companies (17 with aliases)
├── PIPELINE_README.md                   pipeline overview
├── consolidate_scores.py                scoring engine
├── export_dashboard_data.py             data.js builder
├── push_to_github.py                    git push helper
├── _cron_refresh_market_caps.py         daily market cap refresh
├── _weekly_strict_filter.py             weekly cron noise filter
├── _orphan_signal_check.py              signal-to-universe integrity check
├── _apply_orphan_cleanup.py             (historical, one-shot cleanup)
├── calibration_deep.py, calibration_deep2.py    (calibration regen)
├── domain_calibration.json              scoring calibration
├── optimized_domain_params.json         calibration params
├── consolidated_scores.json             cached scoring output (regen'd nightly)
├── current_data.json                    (historical reference, Apr 28 — stale)
├── universe_expansion_queue.md          current backlog
├── company_universe.json.bak_*          backups
└── research/
    ├── admin_signals.json               198 curated signals (ships to deploy repo)
    ├── admin_signals.json.bak_*         backups
    ├── all_signals.json                 3,656 raw research signals
    ├── backfill_candidates.json         (historical)
    ├── backfill_summary.md              (historical)
    ├── tech_expansion_candidates.json   (historical)
    └── tech_expansion_summary.md        (historical)

/home/user/workspace/hr-tracker-worker/            (Cloudflare Worker source, NOT in this repo)
├── worker.js                            request-submission proxy + admin UI
├── wrangler.toml                        Worker config
└── README.md                            deploy instructions

/home/user/workspace/cron_tracking/                (Cron artifacts, NOT in this repo)
├── 1c8bfaa8/                            daily cron artifacts + rotation state
├── 64f9ba23/                            weekly cron artifacts
└── request_notify/                      request-notify cron state
```

End of HANDOFF.md.
