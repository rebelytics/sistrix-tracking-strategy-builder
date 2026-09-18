---
name: sistrix-tracking-strategy-builder
description: 'SISTRIX companion to the platform-agnostic `ai-visibility-tracking-strategy-builder` skill — build or refine an AI visibility tracking strategy on SISTRIX''s custom prompt tracking (AI trackers), with this skill holding everything SISTRIX-specific: the tag-only grouping model and the tag-position convention, the semicolon CSV import format and its per-market/per-upload constraints, the update-quota budget formula and the daily-vs-weekly trade-off, competitor configuration, and how the Analyse step reads tracker data through the SISTRIX API/MCP. Load whenever the user mentions a SISTRIX AI tracker, SISTRIX custom prompts, "what prompts should we track in SISTRIX", a SISTRIX prompt CSV, SISTRIX AI visibility budget or quota, or an AI visibility strategy for a brand that uses SISTRIX. Requires the core skill `ai-visibility-tracking-strategy-builder` loaded alongside; companion to `sistrix-mcp` (recommended; API and MCP mechanics).'
version: 1.2.0
license: CC-BY-4.0
origin: https://github.com/rebelytics/sistrix-tracking-strategy-builder
maintainer: Eoghan Henn / rebelytics (eoghan@rebelytics.com)
---

# SISTRIX Tracking Strategy Builder

**Created by Eoghan Henn / [rebelytics.com](https://www.rebelytics.com)**

The **SISTRIX companion** to
[`ai-visibility-tracking-strategy-builder`](https://github.com/rebelytics/ai-visibility-tracking-strategy-builder)
— the platform-agnostic core that holds the methodology: the intake
rings, the allocation methods, the cardinality rule for containers versus
tags, the three-way brand-mention split, the prompt authoring rules, the
disposition framework, the pattern library, the Phase B stakeholder rules
and the quality gates. This skill holds what is true of **SISTRIX's custom
prompt tracking** specifically: how a tag-only grouping model carries the
core's containers, the CSV import contract, the update-quota economics
that decide how many prompts the plan can carry, competitor configuration,
and how tracker data is read back for Analyse.

**Load both.** Every section here is a "§N — SISTRIX implementation" part
of a core section with the same number. Read the core section first, then
this skill's part. Running this skill without the core produces an import
file with no strategy behind it.

**Licence:** Released under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Share and adapt
for any purpose with appropriate credit.

**Feedback & Support:** If the methodology here proves wrong or incomplete,
or you hit SISTRIX behaviour this skill doesn't cover, log it and open an
issue on the skill's
[public repository](https://github.com/rebelytics/sistrix-tracking-strategy-builder),
or contact Eoghan Henn via [rebelytics.com](https://www.rebelytics.com).
Methodology findings that are not SISTRIX-specific belong on the core
skill's repository. If a failure is the agent not following this skill,
acknowledge and correct it rather than blaming the methodology.

---

## 1. What this skill is (and isn't)

**Is:** The SISTRIX half of a two-skill workflow that takes a brand from
"we have a SISTRIX AI tracker" — or "our tracker's prompt set no longer
fits our strategy" — to a measured, well-structured tracking strategy. The
workflow, its loop (Intake → Strategy → Write → Analyse), its deliverables
and its gates are defined in the core skill (§1, §6, §16 there); this
skill supplies the SISTRIX mechanics each phase needs.

**The write path is an import file, not an API.** At last verification,
custom prompts enter a SISTRIX AI tracker through a CSV upload in the UI
(§12 below), so the core's §14.16 "configuration deliverable for
platforms without a write API" applies: the validated CSV per market is
the write, and a re-read of the tracker's prompt list through the API is
the verification. Re-check whether SISTRIX has since added a write
endpoint before assuming this.

**Two SISTRIX products, one of them in scope.** SISTRIX offers a
pre-built AI Visibility Index over a very large fixed prompt pool (not
customisable) and **custom prompt tracking** inside AI trackers
(user-defined prompts). This skill covers the custom prompt tracking.
The fixed pool is an intake source (§8), not the thing being built.

**Isn't:** A SISTRIX API/MCP reference. Tool mechanics, response-shape
quirks and error patterns live in the companion
[`sistrix-mcp`](https://github.com/rebelytics/sistrix-mcp) skill.
Strongly recommended to load alongside; see §5.

---

## 2. When this skill fires

Trigger when the user wants to set up, rationalise, review or present a
tracking strategy **on SISTRIX** — the trigger list in core §2 applies,
with "the platform" read as SISTRIX. When the platform is not SISTRIX,
load the core and the matching companion instead.

---

## 3. Cross-cutting principles — SISTRIX notes

The principles are core §3. One SISTRIX fact attaches:

**§3.1 instrument separation.** SISTRIX has no native branded /
non-branded concept, and the tracker's aggregate competitor view
(`ai_tracker view=competitors`) is computed across **all** prompts, so the
brand the tracker is built around is structurally advantaged in it and it
**cannot be tag-filtered through the API**. The brand-mention split is
carried by a tag on every prompt (§9.7 below), the brand's own discovery
presence is computed from the per-prompt view filtered by that tag, and
any competitor ranking quoted from the aggregate view carries the caveat.
`sistrix-mcp` "AI visibility — branded vs discovery" is the read-side
rule; read it before reporting any number.

---

## 4. Core principles — SISTRIX implementation

### 4.8 Containers and tags — SISTRIX implementation

SISTRIX exposes **one multi-valued field only**: tags. There is no
single-valued container. The core's cardinality rule (§4.8, §9.4) therefore
resolves to the **"one multi-valued field only" shape**: the budget is
global, categories are analytical labels, and the partitioning dimension
has to be carried **by convention** — the commercial category goes in the
**first tag position on every row**, held absolutely, because nothing in
the format enforces it (§9.4 below). Market is not a tag: language and
country are set per upload and apply to the whole file (§12) — and both
menus are shorter than the tool's market coverage suggests, with the CSV
import narrower than the manual panel, so the market set is checked
against both lists before prompts are authored (§9.2).

### 4.9 Engine coverage — SISTRIX implementation

The engines a prompt is tracked on are selected **at upload time** for the
whole file, not per prompt. At last verification the selectable set was
ChatGPT, Perplexity, Google AI Overview and Google AI Mode; confirm the
current list in the upload dialog. Every additional engine multiplies
quota consumption (§9.1 below), so the core's "match engines to the
audience" rule is a budget decision here, not only a coverage one.

For an **existing** tracker the API does not return which engines it runs
on. Divide its reserved updates by its prompt count: the resulting
per-prompt monthly rate pins the engine count and frequency combination
that produced it (§9.1). Confirm the names themselves in the UI.

### 4.11 Sign-off is the audit trail — SISTRIX implementation

Because the write is a CSV upload, the sign-off artefact and the validated
CSV are the audit trail, and verification is a re-read of
`ai_tracker view=prompts` after upload (§12 below).

### 4.12 Fresh prompts need time — SISTRIX figure

A prompt's first results arrive with the tracker's next scheduled run;
the run frequency is chosen per tracker (daily or weekly, §9.1). Plan the
first Analyse for after the first *complete* run at the chosen frequency
— a weekly tracker needs a week, not a day.

---

## 5. Relationship to other skills

| Skill | Role |
|---|---|
| [`ai-visibility-tracking-strategy-builder`](https://github.com/rebelytics/ai-visibility-tracking-strategy-builder) | **Core — required.** Holds the methodology this skill implements for SISTRIX. Load it first; every section here is a "SISTRIX implementation" part of one of its sections. |
| [`sistrix-mcp`](https://github.com/rebelytics/sistrix-mcp) | **Companion — strongly recommended.** The SISTRIX API/MCP field guide: tool inventory, the `ai_*` tools and their response shapes, the tracker-prompt dedupe rule, the scope limit of `ai_prompt_answers`, the branded-vs-discovery reporting rule. This skill's Analyse mechanics (§13 below) are pointers into it. |
| External-data skills (varies) | Any skill the user has for pulling search-console data, analytics data, crawl data or AI-citation data. This skill consumes their output. |
| Brand-context skills (varies) | Any skill holding accumulated knowledge about the brand being tracked. Always check before asking the user. |

---

## 6. Workflow overview

Core §6 applies unchanged. The SISTRIX facts it needs: project state is
read from `ai_tracker view=prompts` (§8 below); the Write step produces
one CSV per language–country pair and the user uploads it (§12); the
pacing figure is the tracker's run frequency (§4.12).

---

## 6a. Section map — where to read what

The core skill's section map resolves every § to a core file; this map
resolves the same § to its SISTRIX part. Read the core file for a section
first, then the SISTRIX file. Load triggers are mandatory.

| File | Sections | Load trigger |
|---|---|---|
| `references/sistrix-implementation.md` | §7 intake-state fields · §8 tracker reads and the fixed-pool intake source · §9.1 quota budget formula, the frequency trade-off and the subsample test for daily-vs-weekly · §9.2 the country and language lists and when to check them · §9.3 competitor configuration · §9.4 tag conventions and tag naming · §9.7 the brand-mention tag · §12 CSV contract, upload, verification · §13 Analyse reads, including the UI route to per-answer visibility · §15 SISTRIX gate items | After the corresponding core file, before the first SISTRIX read of an Intake step, before sizing any prompt budget, before fixing the market scope, before authoring or validating a CSV, before deciding or defending a run frequency, and before any Analyse read |

---

## 16. Output deliverables

Core §16 applies, in its §14.16 form: the mandatory deliverable is one
validated prompt CSV per language–country pair plus a handover note
carrying the upload settings (language, country, engines, frequency) and
the two reporting instructions from core §14.16; the tracker configured
from those files is the live result. Working artefacts and the optional
Phase B deliverable are as in the core.

---

## 17. Contributing back

Open an issue on
[github.com/rebelytics/sistrix-tracking-strategy-builder](https://github.com/rebelytics/sistrix-tracking-strategy-builder)
when something lands that would help the next person — a changed upload
contract, a quota rule, a new engine, a pattern that only exists on
SISTRIX. If the pattern would hold on any platform, it belongs on the core
skill's repository. Don't fork in-session: feedback → issues → considered
patches → version bump is the path.

## 18. Licence & attribution

**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
Reuse, adapt, redistribute — just keep attribution.

> SISTRIX Tracking Strategy Builder, maintained by Eoghan Henn
> (www.rebelytics.com),
> github.com/rebelytics/sistrix-tracking-strategy-builder.

**Not affiliated with SISTRIX.** SISTRIX has not reviewed or endorsed
this skill. Platform facts here were observed at last verification and
carry re-check instructions; SISTRIX iterates, so verify before relying
on any of them.
