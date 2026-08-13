# Claude Code Instructions — NKT MI Agent (Graph-only)

You are extending an existing, working Python pipeline. Do not rewrite it from
scratch. Preserve its style: stdlib-first, additive SQLite migrations, one
standalone CLI per stage, data on stdout and progress on stderr, defensive
parsing of model output.

---

## 1. REPO STATE

Present: `run.py` · `ingest.py` · `extract.py` · `dedupe.py` · `score.py` ·
`publish.py` · `sources.json` · `relevance_prompt.txt`, plus committed `.pyc`
files to remove.

Absent: `articles.db`, `.env`, `requirements.txt`, tests, `newsletter_prompt.txt`.

**There is no database.** The first run is a cold start against empty state.

**`run.py` is in good shape — do not rewrite it.** It already has PID-based
locking with stale-lock recovery, `--plan`, `--reset --yes`, stage selection
(`--only/--from/--to/--skip`), per-stage env checks that fail before doing work,
and `PYTHON = sys.executable`. One real bug: `exit_code` is assigned inside the
`try` but read after the `finally`, so it is unbound if a stage raises.
Initialise it to 1 before the `try`.

The user has already placed three replacement files in the working folder,
overwriting the repo versions:
- `relevance_prompt.txt` — company tiering, criticality, award segments
- `sources.json` — 96 entries (13 scrape, 48 rss, 35 gnews; 58 enabled)
- `newsletter_prompt.txt` — drives newsletter assembly

Never commit `.env`, `articles.db`, or anything containing a Google Alert feed
URL (unauthenticated access tokens), a SharePoint ID, or an API key.

---

## 2. ENVIRONMENT — VERIFIED, TREAT AS FACT

- **Host**: Azure Arc-enabled RHEL 9 VM on VMware, uptime one month, always on.
  The user has **no sudo rights**.
- **Python 3.9** — RHEL 9 default, confirmed by `cpython-39` bytecode. No
  `match`, no `X | Y` runtime unions, no `tomllib`. `azure-identity` supports it.
- **Path** `/data/elena/mi-agent` on a 100 GB volume. The **root filesystem is
  96% full** — never write outside `/data`.
- **Virtualenv** `/data/elena/mi-agent/.venv`; invoke as `.venv/bin/python`.
- **Managed identity works**, for both `cognitiveservices` and `graph` scopes,
  **including from a stripped non-interactive environment** — cron will work.
  The user is in the `himds` group.
- **Azure OpenAI**: `https://<resource>.openai.azure.com/`, deployments
  **`gpt-5-mini`** and **`gpt-5.5`**.
- **Graph write to SharePoint verified**: `Sites.Selected` granted including the
  per-site grant; `POST .../lists/{list}/items` returned **201**.

Graph works today, so there is **no webhook transport and no transport
abstraction**. Do not build `transport.py`, `transport_webhook.py`, or an
`MI_TRANSPORT` switch. One backend, called directly.

Power Automate keeps two jobs, both built by the user, both outside this
codebase: publishing an approved item to the MI news site, and emailing the
analyst when a draft file appears. **Never send email from Python.**

### SharePoint objects — freshly created, empty

- Site: **Data&AI** (`https://mynkt.sharepoint.com/sites/dataai`)
- List: **`MI News Items v2`** — brand new, only the default Title column
- Library: **`Newsletter Drafts`** — brand new, empty

---

## 3. STEP 0 — CONNECTIVITY GATE (write no other code until this passes)

A previous attempt returned **404** on
`/openai/deployments/gpt-5-mini/chat/completions?api-version=2024-10-21`.
Cause unknown. Determine it empirically. Two candidates:

- **api-version too old** for GPT-5-family — try `2025-01-01-preview` and any
  newer version the resource reports.
- **`max_tokens` rejected** — GPT-5-family models use
  `max_completion_tokens`; the old name can fail confusingly.

1. Read `AOAI_ENDPOINT` from `.env` or ask the user.
2. Token:
   ```python
   from azure.identity import ManagedIdentityCredential
   tok = ManagedIdentityCredential().get_token(
       "https://cognitiveservices.azure.com/.default").token
   ```
   On `CredentialUnavailableError`: likely `himds` group not applied to the
   current shell — tell the user to run `newgrp himds`. Do not work around it.
3. `GET {AOAI_ENDPOINT}/openai/deployments?api-version=2023-03-15-preview`
4. Each deployment × each candidate api-version × each token parameter name —
   attempt a minimal completion, record what returns 200.
5. Test JSON mode (`response_format={"type":"json_object"}`) on the working
   combination. Scoring depends on it.
6. Report the matrix to the user before writing anything else.

Record in `.env`: `AOAI_API_VERSION`, `AOAI_TOKEN_PARAM`, `MI_MODEL_SCORE`
(recommend `gpt-5-mini`), `MI_MODEL_NEWSLETTER` (recommend `gpt-5.5`).
**Nothing about the model is hardcoded anywhere.**

---

## 4. STEP 0b — DISCOVER THE SHAREPOINT IDs

Provide `.venv/bin/python sharepoint.py --discover` which, using `graph.py`:

- resolves the site: `GET /sites/mynkt.sharepoint.com:/sites/dataai` → `SP_SITE_ID`
- lists `/sites/{id}/lists`, finds `MI News Items v2` → `SP_LIST_ID`
- lists `/sites/{id}/drives`, finds `Newsletter Drafts` → `SP_DRAFTS_DRIVE_ID`

Print all three and offer to append them to `.env`. If any is not found, say
which and stop — do not create SharePoint objects the user did not ask for.

---

## 5. STEP 0c — CREATE THE LIST SCHEMA

The list is new and has only `Title`, so `--check-schema` will correctly report
every other field missing. That is expected, not an error.

`sharepoint.py --check-schema` reads the list's columns via Graph and reports
what is missing, with the type each should be.

`sharepoint.py --create-schema` creates them
(`POST /sites/{id}/lists/{id}/columns`). Choice columns take their allowed
values from `taxonomy.py`, so they cannot drift from what the model emits.
**Ask the user to confirm before running it.**

Types: text for names and free text; note (multi-line) for Summary,
NKTImplication, CriticalReason, NewsletterLine, AnalystNotes; number for
RelevanceScore; boolean for IsNKT, Critical, PaywalledSuspect, Posted,
CriticalApproved, InNewsletter; dateTime for PublishedDate; choice for
Relevance, Priority, PrimaryCategory, AwardSegment, PublishSection,
CompanyTier, MarketSignal, InformationStatus, Status.

---

## 6. GLOBAL

`requirements.txt` pinning `azure-identity`, `trafilatura`, `selectolax`,
`py3langid`, `pdfminer.six`, `python-dateutil`.
`.gitignore`: `__pycache__/`, `*.pyc`, `articles.db*`, `.env`, `*.lock`,
`*.log`, `heartbeat.jsonl`.

**`taxonomy.py` — single source of truth.**
- `CATEGORIES`: `Project Awards`, `New Factory/Factory Expansions`,
  `M&A/Divestment News/JV`, `R&D/Product Management`, `CLV/Installation News`,
  `Legal/Security`, `Sustainability/HSE/ESG`, `Investor Relations/Finance`,
  `TSO/Developers`, `Policy/Govt`, `Market Reports/Others`
- `AWARD_SEGMENTS`: `Transmission`, `Grids`, `IAC/MV/LV`
- `PUBLISH_SECTION_MAP`: Project Awards → `Project Awards News`;
  M&A/Divestment News/JV → `M&A News`;
  CLV/Installation News → `Maritime Asset News (CLV & CLB)`;
  New Factory/Factory Expansions → `Factory (Expansion) News`;
  R&D/Product Management → `R&D/Product Development News`;
  all others → `Other News`
- `COMPANY_TIERS`: company plus aliases and local-script names →
  `NKT`|`Tier 1`|`Tier 2a`|`Tier 2b`, transcribed from the tier table in
  `relevance_prompt.txt`. Reporting and filtering only — the model assigns
  `company_tier` independently.

**`azauth.py`** — `ManagedIdentityCredential` wrapper, token cache (refresh
under 5 min to expiry), both scopes. `MI_AUTH=managed_identity|api_key`
(api_key preserves local-model testing). On `CredentialUnavailableError`, exit
with a hint covering `himds`, `IDENTITY_ENDPOINT`/`IMDS_ENDPOINT`, `NO_PROXY`.

**`graph.py`** — thin client: token from `azauth`, JSON helper, retry on
429/5xx honouring `Retry-After`, paging on `@odata.nextLink`.

### `.env`

```
MI_AUTH=managed_identity
AOAI_ENDPOINT=https://<resource>.openai.azure.com
AOAI_API_VERSION=              # Step 0
AOAI_TOKEN_PARAM=              # Step 0
MI_MODEL_SCORE=gpt-5-mini
MI_MODEL_NEWSLETTER=gpt-5.5
SP_SITE_ID=                    # Step 0b
SP_LIST_ID=                    # Step 0b
SP_DRAFTS_DRIVE_ID=            # Step 0b
MI_MIN_PUBLISH_SCORE=40
MI_SCORE_HEADLINE_ONLY=true
NL_THRESHOLD=20
NL_COOLDOWN_DAYS=10
NL_CRITICAL_MIN_ITEMS=0
NO_PROXY=localhost,127.0.0.1
```

`run.py`'s `load_env_file()` already reads this without overriding real shell
variables — keep that, it is how the user overrides a threshold for one run.

---

## 7. FILE BY FILE

**`run.py` — KEEP, four changes.**
(a) Rename stage `publish` → `sharepoint`.
(b) Add stage `feedback` **first**, so each run reads back yesterday's analyst
    decisions before doing new work.
(c) Subcommand `newsletter-check` calling `newsletter.py`, with its own lock
    file so it cannot collide with the daily run.
(d) After SUMMARY, append a one-line JSON heartbeat to `heartbeat.jsonl`.
    Fix the unbound `exit_code`.

**`ingest.py` — extend.**
(a) `kind: "gnews"` — entry carries `{query, hl, gl, ceid}`, builds
    `https://news.google.com/rss/search?q=<urlencoded>&hl=&gl=&ceid=`, parsed
    as RSS. Enable the 35 disabled entries once working.
(b) Extend `unwrap_google` to decode `news.google.com/rss/articles/<base64>`;
    on failure keep the Google URL rather than dropping the item.
(c) Optional per-source `source_type` (`official`|`trade`|`aggregator`;
    default scrape→official, rss/gnews→aggregator), `date_format`,
    `company_hint`. New columns.
(d) Parse `published_raw` via `dateutil` (+`date_format`) into `published_iso`;
    empty on failure rather than guessing.
(e) "NKT - press releases" stub: `--inspect`, fill selectors, enable.

**`extract.py` — minimal.**
(a) Backfill `published_iso` from `meta_date`, falling back to `first_seen`.
(b) **Paywall compliance**: plain public GETs only. Never submit credentials,
    never bypass paywalls or consent walls, no circumvention of any kind. When
    a page yields text below the minimum and the note indicates a stub or
    consent wall, set `paywalled_suspect=1`.

**`dedupe.py` — keep the algorithm, two fixes.**
(a) Restrict the closing `dup_checked` stamp to URLs actually compared this
    run. Current bug: textless rows get stamped and are never re-compared
    after `--retry`.
(b) Add `source_type` rank (official < trade < aggregator) to `rank()`,
    between the published check and the scrape tiebreak.

**`score.py` — change.**
Keep the DB flow, `--rescore`, `--report`, `--show`, `--source`, prompt
versioning, defensive `normalise()`.
1. Categories from `taxonomy.py`. Out-of-taxonomy categories are logged AND
   stored in a new `score_warning` column — never silently dropped.
2. Auth via `azauth.py`; bearer-only under managed identity; model from
   `MI_MODEL_SCORE`; api-version and token parameter from `.env`.
3. `response_format={"type":"json_object"}` where Step 0 confirmed it; keep the
   fence-stripping fallback.
4. New fields → columns with safe defaults in `normalise()`: `voltage_classes`,
   `source_type_assessed`, `award_segment` (validated, empty unless Project
   Awards), `company_tier`, `is_nkt` (0/1), `critical` (0/1), `critical_reason`,
   `newsletter_line` (≤200 chars, **from the model — remove the existing
   Python-side composition entirely**). Compute `publish_section` in code from
   `PUBLISH_SECTION_MAP`; never ask the model for it.
5. Headline-only scoring when `MI_SCORE_HEADLINE_ONLY=true`: also score rows
   with status `empty` that have a title, replacing the ARTICLE block with
   `ARTICLE TEXT UNAVAILABLE (page returned no readable text - possibly
   paywalled). Assess from the headline and metadata only.`
6. Add `SOURCE_TYPE:`, `SOURCE_HINT: {company_hint}`, `URL:` to the user message.

**`publish.py` → `sharepoint.py`.**
Keep pending-selection, batching, per-row error handling. Writes via `graph.py`:
`POST /sites/{SP_SITE_ID}/lists/{SP_LIST_ID}/items` per article, storing the id
as `sp_ref`. Defaults flip: publish only `relevant=1 AND score >=
MI_MIN_PUBLISH_SCORE`; `--all` overrides.

Fields: Title=headline, Url, PublishedDate=published_iso, SourceName,
SourceType, Language, Relevance, RelevanceScore, Priority, PrimaryCategory,
Categories, AwardSegment, PublishSection, CompanyTier, IsNKT, Companies,
Competitors, Projects, Countries, VoltageClasses, MarketSignal,
InformationStatus, Summary, NKTImplication, NewsletterLine, Critical,
CriticalReason, PaywalledSuspect, Status="New", Posted=false,
CriticalApproved=false, InNewsletter=false, ClusterId, AnalystNotes (empty).
Multi-value joined "; ". Store `sp_ref` and `publish_error`.

**`feedback.py` — NEW.** The learning loop.

Analyst decisions currently vanish. Rejections of high-scoring items are the
most valuable signal in the system — labelled data produced free, and the basis
for the parallel-run comparison the business asked for.

Reads back via `graph.py` every list item with a known `sp_ref` whose analyst
state changed, recording **alongside** the model's verdict, never overwriting:
`analyst_status` (Approved|Rejected|New), `analyst_notes` verbatim,
`analyst_category` (a correction shows as a difference), `analyst_critical`
(CriticalApproved), `feedback_at`. Additive migration.

`--report`: agreement counts (agree / model-relevant-analyst-rejected /
model-not-relevant-analyst-approved), category agreement with recurring
confusion pairs, agreement by source and by company tier.

`--export-disagreements [path]`: JSONL — headline, URL, model verdict and
reasoning, analyst decision and note. Input to prompt tuning, read by a human.

**Do NOT implement automatic prompt rewriting or fine-tuning.** The loop is
export → a person reads → a person edits `relevance_prompt.txt` → `--rescore`.

**`newsletter.py` — NEW.**

`check`: reads list state via `graph.py` — approved items not yet in a
newsletter, and critical items awaiting inclusion.
**Exclude `is_nkt` items from both the count and the critical trigger.** NKT's
own news publishes to SharePoint but never drives or enters the competition
newsletter.

Trigger when either:
- a non-NKT item has `Critical` AND `CriticalApproved` AND not `InNewsletter`
  — no cooldown; if `NL_CRITICAL_MIN_ITEMS` > 0, also require that many total
  items waiting
- OR non-NKT approved count ≥ `NL_THRESHOLD` **and** days since last draft ≥
  `NL_COOLDOWN_DAYS`

A critical send includes **everything currently waiting**, not just the
critical item. State in a `newsletter_state` table. Otherwise exit 0 within
seconds — most runs do nothing and that must be cheap.

`generate`: group non-NKT items in canonical category order, Project Awards by
`award_segment`, critical items in a top group. Build the JSON payload described
in `newsletter_prompt.txt`, call `MI_MODEL_NEWSLETTER` at temperature 0.2.
**Validate**: every URL in the output must be in the input set (retry once on
violation); per-section counts must match. Then
`PUT /sites/{id}/drives/{SP_DRAFTS_DRIVE_ID}/root:/MI-Newsletter-Draft-{YYYY-MM-DD-HHMM}.html:/content`,
PATCH each included item to `InNewsletter=true`, record the timestamp.

**`tests/` — NEW.** Pytest, network mocked: gnews URL build and base64 unwrap;
date parsing (EN/KO/ZH, garbage→empty); dedup thresholds EN and CJK including
the `dup_checked` regression; `normalise()` with new fields and bad categories;
publish-section map totality; newsletter trigger logic as pure functions
(threshold, cooldown, critical bypass, NKT exclusion); headline-only path;
Graph retry and paging; feedback reconciliation.

**README — NEW.** Quickstart, env table, Step 0 result and reasoning, Arc token
checklist (`himds`, `newgrp`, endpoint vars, `NO_PROXY`), field mapping table,
and the cron lines **commented out** with a note that scheduling is deferred
until manual testing is complete:
```
# 0 8 * * *      cd /data/elena/mi-agent && ./.venv/bin/python run.py >> cron.log 2>&1
# 0 9-18 * * 1-5 cd /data/elena/mi-agent && ./.venv/bin/python run.py newsletter-check >> cron.log 2>&1
```

---

## 8. COLD-START TEST PLAN

The user will run these by hand. **Do not install cron.** Do not run the full
pipeline unattended. Volumes are deliberately capped.

```
1.  run.py --plan                                  # stage order, env check
2.  sharepoint.py --discover                       # IDs into .env
3.  sharepoint.py --check-schema                   # expect: all missing
4.  sharepoint.py --create-schema                  # after user confirms
5.  run.py --only ingest                           # ~900 articles, no backfill
6.  run.py --only extract --extract-limit 100
7.  run.py --only dedupe --dry-run                 # inspect clusters
8.  run.py --only dedupe
9.  run.py --only score --score-limit 5 --dry-run  # eyeball verdicts
10. run.py --only score --score-limit 50
11. score.py --report
12. run.py --only sharepoint --publish-limit 20
13. newsletter-check with NL_THRESHOLD lowered, to force one draft
```

**Expected and correct**: after step 6, ~800 articles remain at status `new`.
They are a reservoir for further testing, not a bug. Do not "fix" it by
processing everything.

---

## 9. OUT OF SCOPE

Excel or CRM writes; personalised newsletters; Azure AI Search; **email from
Python**; Power Automate flow definitions; any webhook transport; installing
cron.

---

## 10. DEFINITION OF DONE

- Step 0 matrix reported; working combination in `.env` and README
- `--discover` found all three IDs; `--create-schema` built the list
- `run.py --plan` shows: feedback, ingest, extract, dedupe, score, sharepoint
- The cold-start sequence above completes, with 20 items visible in the list
- One newsletter draft file in `Newsletter Drafts`, with every URL traceable
  to an input item
- All tests green
- Nothing hardcoded: no model name, api-version, secret, site ID or URL
- `.gitignore` committed; no secrets in the repo
