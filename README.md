# sistrix-tracking-strategy-builder

Open-source **SISTRIX companion** to the platform-agnostic [`ai-visibility-tracking-strategy-builder`](https://github.com/rebelytics/ai-visibility-tracking-strategy-builder) skill, for building or refining an AI visibility tracking strategy on SISTRIX's custom prompt tracking (AI trackers). Load both into any MCP-capable AI agent (Claude, Cursor, Codex, n8n, etc.).

## What this skill does

The core skill holds the methodology — intake rings, allocation methods, the cardinality rule for containers versus tags, the three-way brand-mention split, prompt authoring, the disposition framework, the pattern library, the stakeholder-presentation rules and the quality gates. This skill holds everything SISTRIX-specific: the tag-only grouping model and the tag-position convention, the semicolon CSV import format and its per-market, per-upload constraints, the update-quota budget formula and the daily-vs-weekly trade-off, competitor configuration, and how the Analyse step reads tracker data through the SISTRIX API/MCP. Section numbers are shared: every "§N — SISTRIX implementation" part here extends the core's §N.

## When it triggers

A SISTRIX AI tracker, SISTRIX custom prompts, "what prompts should we track in SISTRIX", a SISTRIX prompt CSV, SISTRIX AI visibility budget or quota, or an AI visibility strategy for a brand that uses SISTRIX.

## What this repo contains

- `SKILL.md` — the SISTRIX notes on the core's principles, the relationship to the core and to `sistrix-mcp`, and the section map.
- `references/sistrix-implementation.md` — the SISTRIX parts of §7–§9, §12, §13 and §15: intake-state fields, tracker reads and API gaps, the quota budget formula and the daily-vs-weekly test, market scope (the country and language lists), competitor configuration, tag conventions and tag naming, the brand-mention tag, the CSV import contract and verification, Analyse reads, gate items.
- `LICENSE` — CC BY 4.0.

## Install

Install **three** skill directories, each with its `SKILL.md` and `references/` travelling together: the core [`ai-visibility-tracking-strategy-builder`](https://github.com/rebelytics/ai-visibility-tracking-strategy-builder) (required), this skill, and the [`sistrix-mcp`](https://github.com/rebelytics/sistrix-mcp) tool companion (strongly recommended). Follow the client-specific path in your MCP-capable agent's skills directory and restart the client.

## Contributing

Open an issue or a PR at [github.com/rebelytics/sistrix-tracking-strategy-builder](https://github.com/rebelytics/sistrix-tracking-strategy-builder). Findings that would hold on any platform belong on the core's repository.

## Credits

- Original author: [Eoghan Henn](https://www.rebelytics.com) / [LinkedIn](https://www.linkedin.com/in/eoghanhenn)
- Not affiliated with SISTRIX.

## License

CC BY 4.0 — see [LICENSE](./LICENSE). Use it, fork it, adapt it, monetise it. Keep the attribution.
