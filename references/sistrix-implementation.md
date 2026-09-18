# SISTRIX implementation parts (§7–§9, §12, §13, §15)

Part of the **sistrix-tracking-strategy-builder** skill (CC BY 4.0 — Eoghan Henn / [rebelytics.com](https://www.rebelytics.com)). This file holds the SISTRIX-specific part of each section; the platform-agnostic rules are in the core skill `ai-visibility-tracking-strategy-builder`, which must be loaded alongside. Section numbers are global across the skill family — the section map in `SKILL.md` says where each § lives.

**Load trigger:** Read the core file for a section first, then the matching part here — before the first SISTRIX read of an Intake step (§8), before sizing any prompt budget (§9.1), before fixing the market scope (§9.2), before authoring or validating a CSV (§9.4, §9.7, §12), and before any Analyse read (§13).

**Contents:**

- 7 — Intake-state fields
- 8 — Intake: tracker reads, known API gaps, the fixed prompt pool as a source
- 9.1 — Prompt volume: the update-quota budget formula, daily vs weekly
- 9.2 — Market scope: the country and language lists, and the check timing
- 9.3 — Competitor configuration
- 9.4 — Tag taxonomy: the tag-position convention and tag naming
- 9.7 — The brand-mention tag
- 12 — Write: the CSV import contract, upload settings, verification
- 13 — Analyse: what the API serves for tracker prompts, what it cannot, and the UI route to per-answer visibility
- 15 — SISTRIX gate items

**Verification note.** Every platform fact below was observed at last
verification. SISTRIX iterates; before relying on a format, a limit or a
tool behaviour, confirm it against the live upload dialog or a control
call, and report drift so the entry can be updated.

---

## §7 — SISTRIX implementation: intake-state fields

Extend the core §7.2 `platform:` block with:

- `tracker_hash` — the AI tracker's opaque id from `ai_tracker_overview`;
  discovered, never guessed (`sistrix-mcp`, Mental model).
- `upload_settings` per market: `language`, `country`, `engines` (list),
  `frequency` (`daily` | `weekly`). These are set in the UI at upload time
  and are invisible in the CSV, so the intake state is the only record.
- `quota`: `monthly_updates`, `reserved`, `user_requested`, `remaining`,
  `last_read_on` — read off the projects list, arbitrated by the billing
  tooltip where the printed figures disagree (§9.1), because the plan's
  capacity is not exposed by the API.
- `brand_mention_tags`: the three tag strings used for the §9.7 split.

---

## §8 — SISTRIX implementation: intake reads

### Tracker reads (Ring 1 — the platform's own data)

- `ai_tracker_overview` — list trackers (hash + name). Call first; hashes
  are opaque and the AI-tracker hash space is separate from Optimizer
  project hashes.
- `ai_tracker view=prompts` — the current prompt inventory with
  `brand_found`, `avg_pos`, `tags`, `models`. **Dedupe by prompt string
  before counting** — at high `limit` the response can repeat prompts with
  `tags:[]` and a reduced model slice (`sistrix-mcp`, `ai_tracker` note).
  Project state: zero rows = new tracker; rows present = existing project,
  the disposition framework (core §10) applies.
- `ai_tracker view=competitors` — the configured competitor roster with
  visibility index, mentions and prompt counts. An **all-prompts
  aggregate** (§3.1 in `SKILL.md`).
- `ai_tracker view=sources_domains` / `sources_urls` — the domains and
  URLs the engines cite for the tracker's prompts; the source-authority
  input for core §13 and §14.

### Known API gaps — say so when asking the user

| Field | What the API gives | Gap |
|---|---|---|
| Plan quota (monthly updates, reserved, user-requested, remaining) | nothing | Read the four figures printed on the projects list, using the billing tooltip to arbitrate disagreements (§9.1), and record them; ask the user for the plan's update allowance if the UI is not reachable. |
| Upload settings (language, country, engines, frequency) of an existing tracker | not returned per prompt | Ask the user, or read the tracker's settings page; record in intake state. |
| Per-prompt answer text for tracker prompts | `ai_prompt_answers` serves only the global prompt pool — a tracker prompt returns `SISTRIX API Error (1000): no result` | Expected, not a defect. Qualitative reading of tracker answers happens in the UI; the API gives presence, position, tags and sources. |
| Per-country entity counts | `country` filter can silently no-op | A per-country value identical to the all-countries aggregate is a fallback artefact, not data (`sistrix-mcp`). |
| Per-answer visibility (executions naming the brand) | `ai_tracker view=prompts` gives `brand_found` (a per-question ever-flag) and `avg_pos`, and **no execution counts** | Not derivable from the API. Take it from the UI by-tags view with the period selector (§13) and state the window. Never substitute a `brand_found` tally (core §13.5). |

### The fixed prompt pool as an intake source

SISTRIX's pre-built AI Visibility Index runs a very large fixed prompt
pool. It is not customisable, but it is an intake source (core §8.5,
"existing AI visibility data"): `ai_entity view=prompts` returns the pool
prompts on which the brand is already mentioned, `ai_entity
view=competition` returns co-occurring entities (a mix of real rivals and
topical concepts — filter the concepts out before treating it as a
competitor list), and `ai_top` gives the global sources and brands in the
category. Use them to seed the discovery prompts and the competitor
shortlist; never copy pool prompts into the tracker unfiltered — they are
not authored against the brand's demand data.

**Tool names in this sub-section are unverified.** `ai_entity` and `ai_top`
were not visible in the MCP's announced tool list at last check; the server
has since consolidated most of its surface into view-parameter tools, and
`ai_check` covers the per-entity readings. Resolve the names against
`sistrix-mcp`'s name-mapping table and the live schema before running any of
these calls — the *intake source* is real whatever the tool is called.

**Query-fanout harvest (core §8.2 Step A).** SISTRIX does not expose the
engines' fanout sub-queries. Run the harvest through any engine or tool
that does, or fall back to the pool's `ai_entity view=prompts` as the
adjacent-question signal.

---

## §9.1 — SISTRIX implementation: the update-quota budget

SISTRIX prices custom tracking in **updates**: one prompt × one engine ×
one run consumes one update from the account's monthly allowance. The
prompt budget the core's allocation methods distribute is therefore a
derived number:

```
max_prompts = total_updates / (engines × runs_per_month)
```

where `total_updates` is the monthly allowance, `engines` the number
selected at upload, and `runs_per_month` ≈ 30 for daily, ≈ 4 for weekly
tracking. Size the budget from this before any allocation; a set authored
to a headcount the quota cannot carry is cut at upload, not at authoring.

**Daily vs weekly.** Daily tracking costs roughly seven times the quota of
weekly tracking for the same prompt set. Unless intra-week movement is
strategically important (a launch, a campaign, a reputation event),
recommend weekly for custom prompts: the saving buys engines, markets or
seasonal coverage on the same allowance. State this in the sign-off as an
override-able recommendation (core §4.5), not a rule.

**To test whether daily is worth it, run a subset daily for two to three
weeks and subsample it; never compare daily markets with weekly markets.**
The tempting test — leave some markets on daily and others on weekly, then
compare — is confounded and cannot answer the question: frequency is fully
entangled with market and with prompt set, so any difference between the two
groups is equally explained by the markets differing. The daily data already
contains the weekly case. Two reads off one daily subset settle it:

- **Subsample one run per week** from the daily series and compare that
  subsample against all runs. If the weekly subsample tells the same story,
  the extra runs are buying nothing that the reporting uses.
- **Measure the flip rate** — how often a prompt × engine result changes
  between consecutive runs. A low flip rate means daily is resolving noise
  the deliverable will smooth away anyway; a high one means weekly would
  miss real movement.

Pick the daily subset to be representative of the set as a whole (a slice of
each category, not one category), and run it long enough that the weekly
subsample has three or more points. Core §12.7 holds the platform-agnostic
form of this rule and the uneven-evidence consequence of having already run a
mixed-frequency setup.

**Reading the quota — the projects list first, the tooltip to arbitrate**
(verified at last check). The projects overview prints four labelled figures
directly: package allowance, reserved, user-requested and remaining. That
is the cheap read and it comes first. The billing tooltip on hovering the
quota values reads `consumed + forward_reserved + remaining = total` and
remains the arbiter when the printed figures disagree with each other —
they do: in the account observed, allowance minus reserved overstated
remaining by 400. The main dashboard figures also lag or round
inconsistently. Record all four numbers with a date in the intake state,
and re-read after any change — frequency or tracker-count changes take on
the order of tens of minutes to propagate.

**Deriving the per-prompt reservation rate from an existing tracker**
(verified at last check). Where the account already has one configured
tracker, divide its reserved updates by its prompt count and the rate
falls out directly, with no change to the account: in the account
observed, 2,060 / 103 = 20 updates per prompt per month, which is
arithmetically consistent with weekly runs on four engines and
inconsistent with daily runs on any engine count. Free, instant, and it
also cross-checks the engines and frequency an existing tracker is
actually running (`SKILL.md` §4.9), which the API does not return. Try
this before the delta technique below.

**Before/after delta technique** when the plan terms are unclear and no
configured tracker exists to divide: note reserved and remaining; change
one thing (remove one tracker, or switch one tracker from daily to
weekly); wait for the recalculation; note the new values; the delta is the
quota cost of that item, and solves for how many prompts at which
frequency the allowance supports. It costs a change to the account, so it
is the fallback, not the default.

---

## §9.2 — SISTRIX implementation: market scope

**The country list and the language list are two separate constraints, and
the import path is narrower than the manual one** (verified at last check —
re-open both dialogs before relying on the lists; menus change without notice):

| Path | Countries offered | Prompt languages offered |
|---|---|---|
| **Import CSV** dialog | All, Germany, Switzerland, Austria, United Kingdom, Spain, France, Italy, USA | German, English, French, Italian, Spanish |
| Manual **Add prompt** panel | ~50 countries, including Belgium, the Netherlands and Sweden | German, English, French, Italian, Spanish |

Three consequences for the market scope decision, all of which have to be
taken **before prompts are authored**, not at upload:

- **A market whose language is not one of the five cannot be tracked at
  all**, by either path. Core §9.2's "flag and discuss fallback with the
  user before proceeding" applies, and the realistic fallbacks are: track
  that country in one of the five languages and label the prompts as a
  proxy for the market, or drop the market and say so in the sign-off.
- **A supported language in an unsupported import country is
  manual-panel-only**, and the manual panel is expensive: one prompt per
  submission, tags typed by hand each time, and a background-processing
  wait between submissions. Price it in prompt count before committing —
  a market entered this way earns a deliberately smaller set.
- **Bulk markets are the eight import countries.** Anything outside them
  is a cost or an impossibility, so the market list in the strategy is
  checked against both lists above as a pair (language *and* country, for
  the path that will actually be used) before a single prompt is written.
  A set authored for a pair the tool cannot take is wasted work that only
  surfaces at upload.

SISTRIX iterates on both lists; re-read them in the live dialogs at the
start of any market-scope discussion and update the table above.

---

## §9.3 — SISTRIX implementation: competitor configuration

The tracker holds a competitor roster (`ai_tracker view=competitors`).
Configure the core's curated shortlist there; assortment brands (core
§4.6 / §9.3) are **not** roster competitors — carry them as a tag.

**Where it is configured** (verified at last check): **per project** — the
settings page of the individual project, not an account-wide list. That
page holds two things: the **own-brand recognition list**, which takes
multiple spellings and aliases for the tracked brand, and the competitor
slots. Fill the recognition list before reading any mention number: a
brand written more than one way is otherwise counted as itself under one
spelling only. Still **not confirmed**: whether the roster applies to
custom prompts, the fixed pool or both — verify during the competitor
step and record the answer in the intake state.

**The five rendered slots are a default, not a ceiling.** The settings
page pre-renders five competitor slots beside an "Add competitor"
control, and more can be added. Do not argue for a small roster from slot
scarcity — the argument is false and the add control sits next to the
evidence disproving it. The roster stays small on the methodological
grounds the core already gives: a curated handful yields cleaner share of
voice than a sprawl, and assortment brands belong in tags either way. The
recommendation was correct while its stated reason was wrong, which is
exactly the case that does not push back — check the reason, not only the
conclusion.

Sister brands cannot be flagged natively; use a naming convention in the
roster name and note it in the sign-off.

---

## §9.4 — SISTRIX implementation: the tag-position convention

Tags are the only grouping field, flat strings, no hierarchy. The core's
cardinality rule resolves to the "one multi-valued field only" shape, so:

- **First tag = the commercial category** (the core's container), on
  every row, identical spelling across the file and across markets. This
  is the budget container and the dashboard grouping; nothing in the
  format enforces it, so the §15 gate checks it.
- **Second and later tags = cross-cutting dimensions**: the brand-mention
  tag (§9.7, mandatory), funnel stage, intent, seasonality, geographic
  intent, price tier, `regulatory:restricted`, `signal:` dispositions.
  Use the core's namespaced vocabulary (`funnel:`, `intent:`, `type:`) so
  additive tags cannot collide with the one-and-only-one dimensions.
- **Hard ceiling: 250 tags per project** (verified at last check). The
  core's 15–25 tag guidance sits far inside it for a single-market
  project, so the ceiling only bites where tag count multiplies —
  many markets in one project, or per-brand and per-category tags used
  together. It does not bite at all where each market's category tags are
  held identical by convention (§9.4 above), which is one more reason to
  hold that convention.
- **Sprawl discipline** (core §9.4): a tag applied to nearly every prompt
  discriminates nothing.
- **Tag naming is case- and whitespace-sensitive in the export**; hold one
  spelling per tag across the whole file and across markets — the same
  category must carry the same tag in every language file so cross-market
  comparison works.
- **Never put an ampersand in a tag name — spell it "and"** (verified at
  last check). `&` is legal in the CSV and in the data model, and the tag
  is created normally, but the UI's multi-select tag filter
  HTML-entity-encodes the ampersand before URL-encoding it, so any tag
  containing `&` matches nothing and is **silently dropped** from the
  selection. Observed: a fourteen-tag "everything except the branded tag"
  selection on a sources report applied only the six ampersand-free tags
  and returned roughly 45% of every unfiltered count — a smaller,
  plausible number, not an error. So write `corporate and m-and-a`,
  `brand-and-competitive`, `banking-finance`; keep tag strings to letters,
  digits, spaces and hyphens generally, since a name that is legal in the
  data model can still be unreachable through the interface. The read-side
  tell and the single-tag URL workaround are in `sistrix-mcp`
  ("AI visibility — branded vs discovery").
- **Get tag names right at authoring time; a later rename costs the
  tag-level trend.** SISTRIX has no rename function. The working procedure
  is add-new-tag then remove-old-tag over the tag's prompt list, and in the
  removal dialog every tag present in the selection is pre-ticked, so a
  careless confirm strips the prompts' other tags too. Worse, tag-level
  history does not follow: the old tag survives in the by-tags view as an
  unnamed row carrying its accumulated executions, and the per-tag time
  series is split at the rename date (per-prompt history is untouched).
  Renaming a handful of tags is also tens of manual browser actions. This
  is why the naming rules above are a Write-step gate (§15) and not a
  cleanup task.

---

## §9.7 — SISTRIX implementation: the brand-mention tag

Carry the core's three-way split as one tag per prompt in a dedicated
namespace: `type:branded` (tracked brand named), `type:competitor`
(another brand named, tracked brand not), `type:unbranded` (nobody named).
Exactly one per row. Reporting then works as in §13 below: the brand's
own presence is computed from `ai_tracker view=prompts` filtered by this
tag after deduping; the aggregate competitor view cannot be filtered by
it, so any competitor ranking from that view is quoted with the caveat.

---

## §12 — SISTRIX implementation: Write = CSV upload

### The import contract (verify against the live upload dialog)

**CSV is the contract, not one option among two** (verified at last check;
a later re-check should retry a small XLSX before ever recommending it).
The upload dialog offers "CSV or XLSX". **XLSX does not work**: on test,
the XLSX file broke and the equivalent CSV parsed. Do not recommend XLSX
on the reasoning that a delimiter-free format is structurally immune to
the quoting defects this contract exists to prevent — that reasoning is
sound and the conclusion is wrong. An argument from structure predicts
which failures are impossible, never that a path works at all; where a
verified path and an advertised-but-unverified path both exist, the
verified one is the default and the burden of a live test falls on the
challenger. If a future run does test XLSX, do it as a two-row smoke test
before any real delivery, and report the result.

The dialog's **"List" import path reads from stored SISTRIX list
objects**, not a paste box, so it is not an authoring route either.

- **Delimiter:** semicolon (`;`).
- **No header row** — data starts on line 1.
- **Fields:** `"prompt text";"tag1";"tag2";"tagN"` — prompt first, then one
  or more tags, all values quoted.
- **Tags are flat strings** — no hierarchy, no special characters observed.
- **Per-row tags in the file are authoritative** (verified at last check).
  When an attached file carries tags, the dialog's own tag field
  disappears, so a per-prompt taxonomy is buildable in a single import —
  the tags are not forced batch-level. Country, language, model and
  frequency remain batch-level UI settings.
- **900 characters per prompt** (verified at last check). Check the longest
  row before upload.
- **Language and country are not in the CSV** — selected in the UI at
  upload, applying to the whole file. **Engines** and **run frequency**
  are also chosen at upload, not per prompt.
- **The dialog's country and language menus are short, and they are not the
  same menus the manual panel offers** (verified at last check): eight
  countries plus "All" (Germany, Switzerland, Austria, United Kingdom,
  Spain, France, Italy, USA) and five languages (German, English, French,
  Italian, Spanish). §9.2 above holds the full comparison and the
  strategy-time check; by the time a CSV exists, an unsupported pair is
  already wasted authoring.
- **Record the run frequency per file in the handover.** The UI gives no
  way to read an existing batch's frequency back afterwards, so the
  handover note and the intake state are the only record (§7).
- **One CSV per language–country pair.** Name files by market
  (`<brand>-prompts-en-gb.csv`, `<brand>-prompts-de-de.csv`); the same
  file tracks across every engine selected at upload.
- **Encoding:** UTF-8; check for character corruption in non-English
  files.
- **No semicolons inside prompt text** — they break the parser. Rephrase
  the prompt rather than escaping.

Example rows (illustrative, running-shoe retailer):

```
"Best running shoes for flat feet";"Running Shoes";"funnel:consideration";"intent:commercial";"type:unbranded"
"Are Brand X trail shoes good for beginners";"Running Shoes";"funnel:decision";"intent:comparison";"type:competitor"
"How much do running shoes cost";"Running Shoes";"funnel:awareness";"intent:informational";"type:unbranded"
```

### The manual panel as a fallback path

For a language–country pair the import dialog does not offer but the
manual **Add prompt** panel does (§9.2), the prompts go in one at a time.
Budget for it and warn the user before authoring, because three panel
behaviours make it slower than it looks (verified at last check):

- Tags are typed in per prompt — the CSV's per-row taxonomy has to be
  re-entered by hand for every row.
- After each submission the **"Add prompt" button becomes "Filter now"**
  until background processing finishes; the next prompt waits on it.
- The panel **resets on every open** — Germany, German, no models ticked,
  Daily — so country, language, engines and frequency must be re-set for
  each submission. A missed reset silently files the prompt under the
  wrong market or the wrong frequency.

### Wave order

There are no API writes, so the core's wave ordering collapses to: (1)
configure the competitor roster in the tracker (§9.3); (2) upload the
prompt CSV per market with the recorded upload settings; (3) for existing
projects, apply the disposition framework's removals and retags in the
tracker's prompt list first, then upload the additions — the CSV adds
prompts, it does not replace the set.

### Verification (core §12.5)

After upload, re-read `ai_tracker view=prompts`, dedupe, and reconcile:
prompt count per first-tag category equals the approved allocation; every
row carries exactly one `type:` tag; no prompt lost its tags in transit
(a `tags:[]` row that is not a duplicate is an upload defect). Record the
result as the verification log (core §12.6) and the upload settings used.
The first Analyse waits for the first complete run at the chosen
frequency (§4.12).

---

## §13 — SISTRIX implementation: Analyse reads

Read `sistrix-mcp` "AI visibility — branded vs discovery" before reporting
any number, then:

- **Own-brand presence and position**, per cohort: `ai_tracker
  view=prompts`, deduped, split by the `type:` tag and by the first tag
  (category). `brand_found` gives presence; `avg_pos` separates "found"
  from "prominent" — present but mid-pack on the highest-value broad
  prompts is not leading them.
- **`brand_found` is a per-question ever-flag and must not be presented as
  visibility.** It is `true` if the tracked brand was named in **any single
  answer** to that prompt, at any point in the tracker's window — one mention
  in a hundred answers sets it. `view=prompts` returns `brand_found` and
  `avg_pos` and **no execution counts**, so a per-answer share cannot be
  derived from the API at all. A `brand_found` tally is an at-least-once
  count; label it that way or leave it out. Core §13.5 holds the rule and the
  size of the distortion.
- **Visibility for a stakeholder deliverable = the share of answers naming
  the brand, with executions as the denominator, over a stated period — and
  that data is UI-only.** Route, in the tracker's prompts area:
  **prompts → by tags → period selector** (1d / 7d / 14d / 30d / 90d). In
  that view, **"Visibility score" is the share of executions naming the
  brand** and **"Executions" is the denominator**; the **nested prompt rows
  under each tag carry the same two figures per prompt**, which is the level
  the deliverable needs. Extraction notes, each of which has silently
  corrupted a pull:
  - **The period is in the URL path**, not only the selector —
    `…/prompts-tags/tags/tags/lang/all/period/30d`. Read it off the URL and
    put it in the caption; never report a share without the window that
    produced it, and never assume the window a colleague's screenshot used.
  - **Expand the tags and read the nested per-prompt rows.** The tag-level
    row is an aggregate; the prompt-level rows under it are what a
    per-prompt or per-cohort table has to be built from.
  - **After a tag rename the history splits.** The old tag survives as an
    **unnamed ghost row** carrying its accumulated executions while the
    renamed tag starts fresh (§9.4), so a window spanning the rename is
    split across two rows. **Sum mentions and executions per prompt across
    both rows** to restore the full window — summing the *percentages* is
    wrong, because the two rows have different denominators.
  - **Check for a second page of the tags table.** It paginates and says
    "of N pages"; a set read off page 1 alone silently drops whole cohorts,
    and the resulting figure looks entirely plausible.
  Sanity-check the result: executions per prompt should equal engines × runs
  in the window (e.g. four engines × six runs = 24). A prompt whose execution
  count is far off that product was added mid-window or did not run — handle
  it explicitly rather than letting it distort the denominator (core §13.4).
- **Competitor share of voice**: `ai_tracker view=competitors` is the only
  competitor figure the API exposes for tracker prompts, and it aggregates
  across all prompts including branded ones. Report it as "all-prompts
  aggregate, structurally favours the tracked brand"; do not present it as
  a discovery-only ranking. Per-prompt, per-competitor answer text is not
  retrievable for tracker prompts through the API (`ai_prompt_answers`
  serves the global pool only) — qualitative chat reading happens in the
  UI.
- **Sources**: `ai_tracker view=sources_domains` / `sources_urls` for the
  citation map behind "best / top [X]" prompts. These answers are usually
  grounded in a small set of third-party directories, listicles and
  rankings rather than candidates' own sites; where that holds, the
  highest-leverage action is standing within those intermediaries (core
  §11, pattern library; `sistrix-mcp` "Why doesn't our directory rank
  convert" recipe).
- **Per-engine variance**: the `models` field on each prompt row says
  which engines the prompt ran on; a large gap between engines on the
  same prompt is the trigger for engine-level reading (core §13
  principles).
- **Empty vs not mentioned**: a prompt with no result on an engine is a
  different state from `brand_found=false`; keep the two apart in the
  denominator (core §11.22–11.23).

---

## §15 — SISTRIX implementation: gate items

Add to the core's gates:

- **Pre-authoring (§15.1):** every language–country pair in the approved
  market scope checked against **both** SISTRIX lists (§9.2) and recorded
  as import-capable, manual-panel-only (with the extra cost accepted by the
  user) or impossible (with the fallback decided). Run this before a single
  prompt is written — it is the one gate whose failure cannot be fixed
  after the fact, only re-authored.
- **Pre-Write (§15.1):** budget derived from the quota formula with the
  chosen engines and frequency, and the tooltip values recorded with a
  date; upload settings per market recorded; every CSV passes the §15.4
  merged-set gates plus: first tag is a category on every row, exactly one
  `type:` tag per row, no semicolon in any prompt, **no ampersand in any
  tag name** (§9.4 — unfilterable in the UI, and expensive to rename
  later), UTF-8 clean, one file per language–country pair, tag spellings
  identical across files.
- **Pre-Analyse (§15.2):** at least one complete run at the tracker's
  frequency has finished; `ai_tracker view=prompts` deduped before any
  count; the aggregate competitor view's caveat attached to any figure
  taken from it; **no `brand_found` tally reported as visibility** — where a
  visibility figure is needed, the per-answer share is pulled from the UI
  by-tags view with the period recorded (§13), the tags table checked for a
  second page, and any renamed tag's ghost row summed in.
- **Pre-Phase-B (§15.3):** no tracker hash, upload filename or quota
  figure on a stakeholder slide; competitor rankings from the aggregate
  view either excluded or explicitly labelled as all-prompts; **every
  visibility figure expressed as "named in x of N answers" with the answer
  count and the window in the caption** (core §13.5 and the §14 never-list),
  and any per-question at-least-once figure either dropped or labelled as
  such beside the per-answer share.
- **Frequency decisions (§15.2):** no daily-vs-weekly recommendation
  supported by a comparison between markets running at different
  frequencies — the subsample test (§9.1) is the only read that answers it.
  Where the tracker is already running mixed frequencies, the run count per
  market is recorded with the findings and changes to low-run markets are
  limited to breakage (core §12.7).
