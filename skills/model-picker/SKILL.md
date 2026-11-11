---
name: model-picker
description: Choose the right Claude model (Haiku 5.5, Sonnet 5.5, Opus 5.5, Fable 5.2) and effort level for a task, subagent, or API integration. Use when the user asks which model to use, wants to cut cost or latency, is configuring subagent models, mixing models in one system, or migrating a request from one Claude model to another.
---

# Claude model picker

Pick a starting model and effort level, then adjust with evidence.

Every recommendation here comes from Anthropic's published guidance — see [Sources](#sources). When citing advice to the user, link the source. Model lineups and prices change: if a fact matters (price, model ID, default effort), check the live docs before relying on it.

## Default

**Start on Opus 5.5 at medium effort.** It is Claude Code's default and costs half of Fable 5.2 per token. Move from there only with a reason.

- **Down to Sonnet 5.5** — well-scoped everyday work (bug fix, feature against a spec). A clear spec alone is not enough: migrations and perf work can still need Opus or Fable.
- **Further down to Haiku 5.5** — simple, high-volume work: classification, routing, tagging, extraction; cheap subagents.
- **Up to Fable 5.2** — hardest technical work (coding, finance, science), major refactors/migrations, architecture, leading many subagents, long unattended runs, or when Opus at high still falls short.

## Task → starting point

| Task | Model | Effort |
|---|---|---|
| Everyday engineering, debugging, multi-PR features | Opus 5.5 | medium → high |
| Classification, routing, tagging, bulk extraction | Haiku 5.5 | medium |
| Subagents that search, read logs, summarize | Haiku or Sonnet 5.5 | model default |
| Chat / interactive sessions | Sonnet 5.5 (Haiku for simple chat at volume) | low or medium |
| Well-scoped coding (bug fix, spec-driven) | Sonnet 5.5 | medium → high |
| Everyday docs, slides, spreadsheets | Sonnet 5.5 (Opus for complex financial models or huge source piles) | high (medium in Claude Code) |
| Repeated agent tasks (review, investigation, drafting) | Sonnet 5.5 | medium |
| Computer use, vision-heavy loops | Opus 5.5 | medium |
| Major refactors, architecture, orchestrating subagents | Fable 5.2 | medium → high |
| Deep research, scientific/quantitative analysis | Fable 5.2 | medium → high |
| Unsure | Opus 5.5 | medium |

## Model facts

| | Haiku 5.5 | Sonnet 5.5 | Opus 5.5 | Fable 5.2 |
|---|---|---|---|---|
| Input / output per MTok | $0.10 / $0.50 (≤100K prompt); $0.50 / $2.50 (>100K) | $2 / $10 | $4 / $20 | $8 / $40 |
| Cache read per MTok | $0.01 (≤100K) / $0.05 | $0.10 | $0.20 | $0.20 |
| Latency | fastest | fast | moderate | moderate |
| Context / max output | 1M / 128K | 1M / 128K | 1M / 128K | 1M / 128K |
| Thinking | adaptive, can turn off (≤ high effort) | adaptive; `between_tools` option | always on | always on |
| API default effort | medium | high (medium in CC) | medium | high (medium in CC) |
| Fast mode | no | no | yes | yes |

## Tuning order when results fall short

1. Give the model a way to **verify its own work** (test, build, script hitting the endpoint). Usually cheaper than more effort.
2. **Raise effort one level.**
3. **Move up one model**, starting at a *lower* effort than you used on the smaller one.

Notes:
- Effort ≈ thinking time, tool calls, self-checking. It adds latency. It cannot add knowledge the model lacks — that needs a bigger model.
- Past `high`, one more effort level can cost as much as the next model up. Test the bigger model at lower effort instead.
- If outcomes are auto-checkable: run everything at low, re-run only failures at high (beat all-high on both cost and pass rate in Anthropic's tests).
- Need speed without losing capability: fast mode on Opus 5.5 / Fable 5.2 (premium price).

## Cost reasoning

- Compare **cost per finished task**, not per-token price. A pricier model at lower effort can be cheaper per task (e.g. Fable 5.2 at high ≈ Opus 5.5 at max on a hard benchmark for ~half the cost).
- Price the **hardest** tasks in your set; they dominate spend.
- Big context is resent every turn, but mostly bills as cache reads. Fable and Opus share the $0.20 cache-read rate, so on cache-heavy runs their gap is uncached input + output.
- Keep the expensive model's context lean; push verbose reading to cheap subagents.
- Haiku's long-prompt tier (>100K) is 5× its short rate. For long prompts, compare Haiku vs Sonnet per finished task.

## Mixing models

- **Switch at natural breaks.** Prompt cache is per-model, so the first turn on a new model re-pays cache writes. In Claude Code run `/compact` or start fresh with a short plan; on the API, open with a summary.
- **Thinking blocks don't travel down-tier.** Fable 5.2 reads Opus/Sonnet/Haiku 5.5 blocks; Opus 5.5 reads Sonnet/Haiku; Sonnet reads Haiku. Moving down (or off Fable 5.2) silently drops earlier reasoning.
- **Subagents on small models** for work where mistakes are cheap to spot (find files, run tests, read logs). Keep code edits and judgment on the main model. In Claude Code set `model: haiku` / `model: sonnet` in the agent file, or `CLAUDE_CODE_SUBAGENT_MODEL` for the default.
- The built-in Explore subagent now uses the session model. To keep it cheap, define your own `Explore` agent with `model: haiku`. `ANTHROPIC_DEFAULT_HAIKU_MODEL` pins the alias. On the Anthropic API `haiku` → Haiku 5.5 (needs Claude Code ≥ v2.1.293); on Bedrock/Vertex/Foundry it is still Haiku 4.5.
- `opusplan` plans on Opus and edits on Sonnet — the reverse of the advice above. Measure before adopting.
- **Advisor tool (beta):** cheap executor consults a stronger advisor. Gains have been small (≈ what raising effort buys). Measure the consult rate first; check allowed pairings.
- **Lead + workers** for work bigger than one context window (e.g. Opus lead over parallel Sonnet workers: ~⅓ cheaper, same score). Skip for single dependent chains or work that fits in one context.

Example cheap subagent (`.claude/agents/log-reader.md`):

```markdown
---
name: log-reader
description: Reads test output and logs and reports each failing case. Use after a test run.
tools: Read, Grep, Glob, Bash
model: haiku
---

For each failure, report the file, test name and first error line. Do not edit files.
```

## Migration / compatibility checklist

- **Fable 5.2 preserved thinking:** API returns 400 if anything before a Fable thinking block changed (system prompt, tools, earlier messages) unless you opt in to dropping it. Keep history append-only. Claude Code / Agent SDK / Managed Agents handle this.
- **Thinking:** Opus 5.5 and Fable 5.2 can't disable thinking (error). Read content blocks by type — responses may start with a thinking block.
- **Forced tool use:** `tool_choice: any|tool` → 400 on Sonnet/Opus 5.5 and Fable 5.2. Use `auto` + `strict: true` tool + prompt instructions.
- **Computer use:** only via `computer_toolset_20260801` (Claude API, Google-managed Claude on Google Cloud).
- **Haiku 4.5 → 5.5:** replace `budget_tokens` with effort; drop non-default `temperature`/`top_p`/`top_k`; end with a user turn (no assistant prefill).
- **Safety fallbacks:** flagged cyber requests may re-run on Opus 4.8, biology on Opus 5 (Claude Code shows a notice).
- **Multi-model systems:** build every request for the strictest model so steps can move tiers without code changes. If routing is simple (one model per step type), route in code, not with a model.

## Measure before committing

1. Sample real tasks; write an outcome check for each (passing test, correct row count…).
2. Run each candidate at 2–3 effort levels, separate sessions (no shared cache). Record cost from each response's `usage`.
3. Plot score vs spend. Keep a multi-model setup only if it beats every single-model point.
4. Shadow-run the winner on real traffic before cutting over.

Quick check in Claude Code: run one backlog task on two models via `/model`, compare `/usage`.

## How to answer the user

1. Name the model + effort for their task, with a one-line reason from the table.
2. Say what would make them move up or down.
3. If they're integrating via API or mixing models, flag the relevant compatibility items.
4. If cost matters, suggest the measurement loop above rather than trusting list prices.

## Sources

Primary source:

- Addy Osmani, ["Which Claude model should you build with? Fable, Opus, Sonnet, or Haiku"](https://claude.dev/blog/which-claude-model-should-you-build-with), claude.dev Blog (Anthropic), Nov 9, 2026.

Anthropic docs referenced by that article:

- [Choosing a model](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model) — Claude Platform docs
- [Models overview](https://platform.claude.com/docs/en/models/overview) — model IDs, pricing, context windows
- [Effort](https://platform.claude.com/docs/en/build-with-claude/effort) — effort levels and their trade-offs
- [Claude Code model configuration](https://code.claude.com/docs/en/model-config) — aliases, `opusplan`, `ANTHROPIC_DEFAULT_HAIKU_MODEL`
- [Claude Code subagents](https://code.claude.com/docs/en/sub-agents) — per-agent `model:` and `CLAUDE_CODE_SUBAGENT_MODEL`
