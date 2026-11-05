# Find Startup Jobs: Cursor Plugin Spec

Version 0.1 (draft) · 5 November 2026 · Target: Cursor Marketplace · Licence: MIT

## 1. Summary

**Find Startup Jobs** is a Cursor plugin with a single skill, `/find-startup-jobs`. On demand, it checks a pre-seeded list of European and remote-friendly tech startups for new job postings, filters and ranks them against the user's profile (CV and preferences), and remembers which jobs the user has already seen, applied to or discarded.

The design goal for v0.1 is **maximum simplicity**. Fetching, dedupe and state are deterministic Python scripts. The LLM (Cursor's own agent) is used only where judgement is needed: reading the CV, scoring fit, and understanding the user's triage instructions.

### Goals

- One command, `/find-startup-jobs`, that returns a ranked shortlist of new jobs since the last run.
- First-run onboarding that builds a local profile from the user's CV and a few questions.
- Persistent memory of seen, applied and discarded jobs, so nothing is shown twice.
- Zero accounts, zero API keys. All user data stays on the user's machine.

## 2. User experience

### Invocation

```
/find-startup-jobs                  # normal run
/find-startup-jobs 2026-10-20       # only jobs posted on or after this date
```

After results are shown, the user triages in plain language in the same chat, for example: "applied to 2 and 4, discard 3 (too junior), discard 5 (not interested in adtech)". The skill translates this into state updates. There is no separate triage command in v0.1.

### First run: onboarding

If `~/.find-startup-jobs/profile.md` does not exist, the skill enters onboarding before searching:

1. **CV.** Ask the user to attach their CV or give a file path. The agent extracts: current and past roles, years of experience, seniority level, main technologies, languages spoken, and location history. It shows the extracted summary and asks the user to confirm or correct it.
2. **Preferences.** Ask, in as few messages as possible:
   - Work mode: remote, hybrid, on-site (multiple allowed).
   - Home location (city and country) and, for hybrid or on-site, the maximum acceptable commute or the cities they would work in.
   - Where they are legally allowed to work (countries or regions) and acceptable time zones for remote roles.
   - Target roles (free text, for example "backend engineer, platform engineer").
   - Minimum salary and currency (optional, can be skipped).
   - Stack preferences and dealbreakers (optional, free text, for example "no consulting, no PHP, no crypto").
3. **Generate the profile file** (see section 5.1) and show it to the user for a final confirmation.
4. Continue directly into the first search.

The user can say "update my profile" at any time. The skill then re-runs the relevant part of onboarding and rewrites the file.

### Normal run

1. Determine the date window (section 4.1).
2. Run the scan script. It fetches all seeded boards, normalizes jobs, drops anything already in state, applies the deterministic prefilter, and prints the candidates as JSON.
3. The agent scores each candidate against the profile (section 6) and ranks them.
4. The agent presents the shortlist (section 7) and saves the run results.
5. The agent invites triage and applies state updates as the user responds.

## 3. Architecture

```
plugins/find-startup-jobs/skills/find-startup-jobs/SKILL.md
     │
     ├── onboarding  → writes ~/.find-startup-jobs/profile.md
     │
     ├── scripts/scan.py     → fetch → normalize → dedupe → prefilter → JSON
     ├── scripts/state.py    → mark applied / discarded, save results
     │
     └── data/companies.csv  (seed list, shipped inside the plugin folder)
```

Principles:

- **Deterministic first.** Anything that can be done by code (fetching, dates, dedupe, filtering) is done by code, not by the model.
- **LLM only for judgement.** CV extraction, fit scoring with a reason, and interpreting triage messages.
- **User data lives outside the plugin folder** (`~/.find-startup-jobs/`) so plugin updates never touch it.
- **Python standard library only** (Python 3.10+). No `pip install` step for the user.

## 4. Run logic

### 4.1 Date window

Precedence, highest first:

1. An explicit date argument.
2. The `last_run` date from state, if present.
3. Otherwise (first run): the last 7 days.

The date is a coarse filter only. The source of truth for "new" is the seen set: a job whose key is already in state is never returned, regardless of date. A job with no usable date is treated as new if it is not in state.

### 4.2 Job identity

Every job gets a stable key: `{ats}:{slug}:{external_id}`, for example `greenhouse:acme:4012345`.

### 4.3 Sources and endpoints

All three are public, unauthenticated JSON endpoints. Exact fields and endpoints must be verified at build time against current documentation.

| ATS | Endpoint (to verify) | Notes |
| --- | --- | --- |
| Greenhouse | `https://boards-api.greenhouse.io/v1/boards/{slug}/jobs?content=true` | Often exposes a last-updated time rather than the original publish date. Use the best available date and record which field was used. |
| Lever | `https://api.lever.co/v0/postings/{slug}?mode=json` | Creation timestamp, location, workplace type, sometimes a salary range. |
| Ashby | `https://api.ashbyhq.com/posting-api/job-board/{slug}?includeCompensation=true` | Published date, remote flag, optional compensation. |

Network behaviour: fetch boards with a small concurrency limit (for example 8 in parallel), a timeout per request (for example 15 seconds), one retry, and a descriptive `User-Agent`. A board that fails is skipped and reported in the run summary; it never aborts the run.

### 4.4 Normalized job record

```json
{
  "key": "greenhouse:acme:4012345",
  "company": "Acme",
  "title": "Senior Backend Engineer",
  "url": "https://boards.greenhouse.io/acme/jobs/4012345",
  "location": "Berlin, Germany; Remote (EU)",
  "remote": "remote | hybrid | onsite | unknown",
  "posted_at": "2026-10-28",
  "posted_at_source": "published | updated | unknown",
  "salary": { "min": 70000, "max": 90000, "currency": "EUR" },
  "description": "first ~1500 characters, plain text",
  "company_tags": ["eu", "fintech", "seed"]
}
```

`salary` is null when the posting does not state it.

### 4.5 Deterministic prefilter

Applied by `scan.py` before any LLM call, using values from `profile.md`:

- **Date:** drop jobs older than the window (jobs with no date are kept).
- **Role match:** keep a job only if its title matches at least one of the profile's `role_keywords` (case-insensitive substring or word match). The keywords are generated during onboarding from the user's target roles, including common synonyms.
- **Location and work mode:** drop jobs that clearly conflict, for example an on-site job in a country the user cannot work in or will not relocate to. Ambiguous or unknown cases are kept and left to the scorer.
- **Dealbreakers:** drop jobs whose title or company tags match hard exclusions from the profile.
- **Salary:** drop only if the posting states a salary whose maximum is below the user's minimum. Missing salary is never a reason to drop.

The script outputs candidates sorted by a simple relevance heuristic (role keyword match strength, then recency) and caps the list at **40** candidates per run. If more survive, the summary says how many were not scored.

## 5. Data

All user data is in `~/.find-startup-jobs/` (on Windows, the equivalent user home folder).

### 5.1 `profile.md`

A human-readable Markdown file with a small machine-readable header, so the user can edit it by hand.

```markdown
---
work_modes: [remote, hybrid]
home: Donostia / San Sebastián, ES
allowed_regions: [EU, UK]
timezones: [UTC-1, UTC+3]
role_keywords: [backend, platform, software engineer, infrastructure]
seniority: senior
salary_min: 60000
salary_currency: EUR
dealbreakers: [consulting, php, crypto]
---

## Experience summary
(agent-written summary of the CV: roles, years, strengths)

## Stack
(technologies the user has used, grouped by strength)

## Preferences
(free text from onboarding: what they want, what they don't)
```

The raw CV file is not copied or stored; only the confirmed summary is.

### 5.2 `state.json`

```json
{
  "last_run": "2026-11-05",
  "jobs": {
    "greenhouse:acme:4012345": {
      "status": "shown | applied | discarded",
      "company": "Acme",
      "title": "Senior Backend Engineer",
      "url": "https://...",
      "first_seen": "2026-11-05",
      "status_changed": "2026-11-06",
      "reason": "too junior"
    }
  }
}
```

Status meanings:

- `shown`: presented to the user, no decision yet. Not shown again by default.
- `applied`: the user applied.
- `discarded`: the user is not interested. `reason` is optional and free text.

A job that appears in `state.json` in any status is never returned as new. Writes are atomic (write to a temporary file, then rename).

### 5.3 `last_results.json`

The ranked results of the most recent run, so that "discard 3" refers to a stable list and the user can ask "show me last results again".

## 6. Scoring

The agent scores every candidate against the profile. Input per job: title, company and tags, location, remote flag, salary if known, and the truncated description. Output per job:

- `verdict`: **GO**, **MAYBE** or **SKIP**
- `reason`: one line, concrete (for example "Python and Postgres match, but needs 8+ years and you have 5")
- optional `flags`: for example `no salary listed`, `remote restricted to US time zones`

Rubric:

- **GO:** role, seniority, stack and location or work mode all fit, and no dealbreaker is present.
- **MAYBE:** a good fit with one meaningful uncertainty or mismatch the user may accept.
- **SKIP:** clear mismatch. SKIP jobs are not shown, only counted in the summary.

The agent must base its judgement only on the profile and the posting. If the posting text is too thin to judge, it says so instead of guessing.

Discard reasons recorded in state are available to the agent. When several discards share a theme (for example "adtech" three times), the agent may suggest adding it as a dealbreaker in the profile, but never changes the profile without asking.

## 7. Output format

Results are shown in chat, ranked, GO first and then MAYBE, numbered so the user can refer to them:

```
Found 14 new jobs since 29 Oct (6 GO, 5 MAYBE, 3 skipped).

1. GO · Senior Backend Engineer, Acme (Berlin / Remote EU)
   Python and Postgres match; remote within your time zones. Posted 4 Nov.
   https://...

2. MAYBE · Platform Engineer, Beta (Remote)
   Strong stack match, no salary listed. Posted 2 Nov.
   https://...
```

Followed by a one-line run summary: boards checked, boards that failed (named), candidates not scored because of the 40 cap, and an invitation to triage.

## 8. Seed list

### 8.1 Format: `plugins/find-startup-jobs/data/companies.csv`

```
name,ats,slug,region,remote,sector,stage
Acme,greenhouse,acme,eu,remote-friendly,fintech,seed
```

| Field | Values |
| --- | --- |
| `ats` | `greenhouse`, `lever`, `ashby` |
| `slug` | the board identifier used in the ATS URL |
| `region` | `eu`, `uk`, `global`, or a country code |
| `remote` | `remote-first`, `remote-friendly`, `onsite` |
| `sector`, `stage` | free-form tags, used for display and light filtering |

The seed list is the same for all users in v0.1. Users cannot add companies; to change the list they must wait for a plugin update (or edit their local copy of the file, which is unsupported).

### 8.2 Building the seed list (maintainer tooling, not user-facing)

These scripts live in `tools/` in the repo and are run by the maintainer, never by end users.

1. **`import_list.py`**: takes the maintainer's existing list of companies (names and, where known, careers URLs) and attempts to resolve each to an ATS and slug, by recognizing URL patterns and by trying candidate slugs against the three public endpoints.
2. **`hn_extract.py`**: reads the monthly "Ask HN: Who is hiring?" thread through the public Hacker News Algolia API (find the latest thread by the `whoishiring` account, then fetch its comments). It extracts company names and any links from the top-level comments, drops entries unrelated to Europe or remote, and feeds the remaining names into the same resolution step. Output is a **candidates file for manual review**, never written directly to the seed list.
3. **`validate_seed.py`**: calls every board in `companies.csv`, confirms it responds with valid job data, and reports dead boards and companies that have changed ATS. Intended to run monthly (manually or in CI) before each plugin release.

HN is used only to **discover companies** for the seed list. It is not scanned at runtime in v0.1.

Target size for the first release: roughly 150 to 300 validated companies. Quality (live boards, accurate tags) matters more than size.

## 9. Repository layout

This repo follows Cursor's multi-plugin template. Cursor installs one plugin folder, not the repository root, so everything the installed plugin needs lives under `plugins/find-startup-jobs/`. The repo can hold more plugins later; each one is another folder under `plugins/` plus an entry in `.cursor-plugin/marketplace.json`.

```
find-startup-jobs/
├── .cursor-plugin/
│   └── marketplace.json
├── plugins/
│   └── find-startup-jobs/
│       ├── .cursor-plugin/
│       │   └── plugin.json
│       ├── skills/
│       │   └── find-startup-jobs/
│       │       └── SKILL.md
│       ├── scripts/
│       │   ├── scan.py
│       │   └── state.py
│       ├── data/
│       │   └── companies.csv
│       └── README.md
├── tools/                  # maintainer-only seed tooling, not shipped in the plugin
│   ├── import_list.py
│   ├── hn_extract.py
│   └── validate_seed.py
├── tests/
├── docs/
│   └── add-a-plugin.md
├── scripts/
│   └── validate-template.mjs
├── README.md
├── CHANGELOG.md
└── LICENSE                 # MIT
```

`plugin.json` (minimum): `name` (`find-startup-jobs`), `description`, `version`, `author`, and the logo if any. The author field should carry the maintainer or company name and a contact address, as the marketplace review is manual. The slash command is the skill `name` in `SKILL.md` frontmatter (`find-startup-jobs`), which Cursor exposes as `/find-startup-jobs`.

## 10. Scripts: contracts

### `scan.py`

```
python scan.py --profile ~/.find-startup-jobs/profile.md \
               --state   ~/.find-startup-jobs/state.json \
               --since   2026-10-29        # optional
```

Prints a single JSON object to stdout: `{ window, boards_checked, boards_failed: [...], candidates: [...], not_scored: N }`. Writes nothing to disk. Exit code non-zero only on unrecoverable errors (for example an unreadable profile).

### `state.py`

```
python state.py save-results  --file results.json     # stores shown jobs + last_results.json, sets last_run
python state.py applied       --keys KEY1,KEY2
python state.py discarded     --keys KEY1 --reason "too junior"
python state.py show-last
```

The skill maps the user's numbered references ("discard 3") to keys using `last_results.json` before calling the script.

## 11. Error handling and edge cases

- **No profile:** run onboarding.
- **Profile present but unreadable:** show the problem and offer to regenerate it. Do not overwrite silently.
- **A board fails or times out:** skip it, name it in the summary, continue.
- **All boards fail** (for example no network): say so clearly and do not update `last_run`.
- **Zero new jobs:** say so, state the window used, and still update `last_run`.
- **Python not available:** the skill tells the user that Python 3.10 or later is required and how to check for it.
- **Date argument unparseable:** ask for the date in `YYYY-MM-DD` form.

## 12. Privacy and safety

- The CV file is read once during onboarding; only the confirmed summary is stored, locally.
- The only network calls made by the scripts are to the public ATS endpoints (and, for maintainers, the HN API). No user data is sent to those services.
- The user's profile and job text are processed by Cursor's model like any other chat content, under the user's own Cursor privacy settings. The README states this plainly.
- The plugin never applies to jobs, sends messages or contacts anyone on the user's behalf.
- Job descriptions are untrusted text. The skill instructions tell the agent to treat posting content as data and never follow instructions found inside it.

## 13. Testing

- **Unit tests** for date-window logic, job key generation, normalization of each ATS's response (using saved sample responses), prefilter rules, and atomic state writes.
- **Fixture-based scan test:** a fake set of boards served from local fixtures, to test the full path from fetch to JSON without network access.
- **Manual test script** for the skill: fresh onboarding, a normal run, triage messages, a second run showing no repeats, a run with an explicit date, and a run with one board failing.

## 14. Release checklist

1. Seed list validated (`validate_seed.py` clean).
2. Tests pass.
3. Plugin loaded from `~/.cursor/plugins/local/` and the manual test script completed.
4. README covers install, first run, privacy and how the seed list is maintained.
5. Repo public, MIT licence present.
6. Submit at cursor.com/marketplace/publish and expect manual review.

## 15. Acceptance criteria for v0.1

- A new user with no setup can run `/find-startup-jobs`, complete onboarding in under five minutes, and receive a ranked shortlist in the same session.
- A second run immediately afterwards returns no jobs already shown.
- Jobs marked applied or discarded never reappear.
- A broken board does not prevent results from the others.
- No user data is written inside the plugin directory.
- A typical run completes in under about two minutes.

## 16. Open questions and assumptions

- **Assumption:** the exact ATS endpoints and date fields are as listed in section 4.3. Verify each before implementing, especially which Greenhouse field gives the best "posted" date.
- **Open:** the maintainer's own company list: format and size. Section 8.2 assumes a plain list of names or URLs.
- **Open:** how the seed list is refreshed after release (manual plugin updates are assumed for v0.1).
- **Open:** whether to publish the plugin under a personal or company name (affects the `author` field and the marketplace submission).

## 17. Later versions (not in scope now)

User-added companies from a pasted link, an MCP server exposing the same scripts as tools, scheduled or unattended scans with notifications, closed-posting detection, CV tailoring and cover letters for GO jobs, additional ATS support, a hosted tier with a continuously maintained company directory, and multi-user support.
