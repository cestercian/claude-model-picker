---
name: model-picker
description: Choose the right Claude model (Haiku 5.5, Sonnet 5.5, Opus 5.5, Fable 5.2) and effort level for a task, subagent, API integration, or an app that hosts an agent through the Agent SDK. Use when the user asks which model to use, wants to cut cost or latency, is configuring subagent models, mixing models in one system, or migrating a request from one Claude model to another.
---

# Claude model picker

Pick a starting model and effort, then adjust with evidence. Facts are as of November 2026; if the user's environment (`/model`, `/effort status`, an error, `usage`) disagrees, trust it and say the skill may be out of date.

## Short answer

**Start on Opus 5.5 at `medium`** (its default). Claude Code's default, half Fable 5.2's price, the official starting point. Move only with a reason:

- **Down to Sonnet 5.5** for well-scoped everyday work (bug fix, feature iteration). A clear spec alone isn't a reason: a migration can be fully specified and still hard.
- **Down to Haiku 5.5** for simple high-volume work (classify, route, tag, extract) and side-task subagents.
- **Up to Fable 5.2** for the hardest coding/finance/science work, major refactors and migrations, architecture, leading many subagents, long unattended runs, or when Opus 5.5 at `high` still falls short.

**Effort before model.** More effort buys thinking, tool calls and self-checking (and latency), not knowledge the model lacks. Past `high`, one more level can cost as much as the next model up.

## Model facts

| | Haiku 5.5 | Sonnet 5.5 | Opus 5.5 | Fable 5.2 |
|---|---|---|---|---|
| API ID | `claude-haiku-5-5` | `claude-sonnet-5-5` (Bedrock: `anthropic.claude-sonnet-5-5`) | `claude-opus-5-5` | `claude-fable-5-2` |
| Claude Code alias | `haiku` | `sonnet` | `opus` | `fable` |
| Input / output per MTok | $0.10 / $0.50* | $2 / $10 | $4 / $20 | $8 / $40 |
| Cache read per MTok | $0.01* | $0.10 | $0.20 | $0.20 |
| Context / max output | 1M / 128K | 1M / 128K | 1M / 128K | 1M / 128K |
| Thinking | adaptive; can disable at `high` or below | adaptive; `between_tools` minimum; can't disable | always on; disabling = 400 | always on; disabling = 400 |
| Default effort (API) | `medium` | `high` (`medium` in Claude Code) | `medium` | `high` (`medium` in Claude Code) |
| Fast mode | no | no | yes | yes |

\* Haiku is priced by prompt length: ≤100K tokens as shown; >100K is 5× ($0.50 / $2.50, cache read $0.05), still 1/4 of Sonnet.

All four support `low`, `medium`, `high`, `xhigh`, `max`; levels are calibrated per model, so never carry a level over from an older model. Cache writes: 1.25× input (5-min), 2× (1-hour). Batch API: half price. Fast mode: up to 2.5× output speed at 2× price on Opus 5.5. Claude Mythos 5.2 (`claude-mythos-5-2`) = Fable 5.2 capability, verified orgs only; best for cybersecurity and life-sciences research.

## Task → model + effort

| Task | Model | Effort |
|---|---|---|
| Everyday engineering, debugging, multi-PR features | Opus 5.5 | medium, then high |
| Classification, routing, tagging, extraction at volume | Haiku 5.5 | medium |
| Subagents that search, read logs, summarize | Haiku 5.5 or Sonnet 5.5 | model default |
| Chat, interactive sessions | Sonnet 5.5 (Haiku for simple chat at volume) | low or medium |
| Interactive coding (someone waits) | Opus 5.5; Sonnet 5.5 if edits are scoped and waiting dominates | medium |
| Well-scoped coding to a spec | Sonnet 5.5 | medium, then high |
| Documents, slides, spreadsheets | Sonnet 5.5 (Opus 5.5 for complex financial models or many sources) | high (API default) |
| Repeated agent tasks: investigation, review, drafting | Sonnet 5.5 | medium |
| Computer use, vision-heavy loops | Opus 5.5 | medium |
| Major refactors/migrations, architecture, leading many subagents | Fable 5.2 | medium, then high |
| Deep research, scientific/quantitative analysis | Fable 5.2 | medium, then high |
| Not sure | Opus 5.5 | medium |

## Tuning order

When results fall short:

1. **Give the model a way to verify** (test, build, type-check, script). A check costs one turn; extra effort adds thinking to every turn.
2. **Raise effort one level.**
3. **Move up one model at a lower effort** than you used on the smaller one. Don't wait for a third failure: if Opus 5.5 at `xhigh` hits the same wall twice, switch up, then back once solved.

- **Low, then re-run failures** when outcomes are machine-checkable: Opus 5.5 at `low` + failures re-run at `high` passed ~97% at ~$0.17/task vs 95.3% at $0.29 with everything at `high` (SWE-bench Pro subset).
- **Break-even past `high`**: before pushing Opus 5.5 past `high`, test Fable 5.2 at `high` (FrontierCode 2.0: Fable `high` ≈ Opus `max` at ~half the cost). `high` itself pays off when it saves one retry.
- **Need speed without losing capability**: fast mode on Opus 5.5/Fable 5.2, not lower effort or a smaller model.
- Changing top-level effort mid-conversation breaks the cache (per-message effort, beta, keeps it). Claude Code `/effort` on Opus 5.5 keeps it except on Bedrock, Google Cloud or a gateway.

## Cost TL;DR

- **Price per finished task, not per token.** Terminal-Bench 3.0: Opus 5.5 `high` (58.9%) beat Fable 5.1 `max` (58.0%) on half the tokens.
- **Hardest tasks dominate spend**: on one benchmark, 2 of 20 problems were 43% of cost. Always sample them.
- **Cost = turns × resent context.** Every turn resends the conversation; fewer turns (verification, batched tool calls, one-pass context gathering) is the cheapest saving.
- **Cache hit rate is the biggest input lever**: 2.8M input tokens on Opus 5.5 cost $11.20 uncached vs $1.62 at 90% cached. Opus 5.5 cache read is 1/20 of input; an output token costs 100× a cache read.
- **Output (incl. thinking) is 5× input**, which is why effort moves the bill.
- **Keep the big model's context lean**: Opus and Fable share the $0.20 cache read, so on long runs their gap is only fresh input/output. Push verbose reading to cheaper subagents.
- Typical Claude Code spend: ~$13/dev/active day; under $30/day for 90% of users.

## Mixing models

- **Switch at a natural break**: the cache belongs to the model that wrote it, so the first turn on a new model rewrites it all. `/compact` first or start from a summary; `/model` also sets the default for new sessions, so switch back.
- **Thinking blocks only travel up-tier**: Fable 5.2 reads Opus/Sonnet/Haiku 5.5; Opus 5.5 reads Sonnet/Haiku 5.5; Sonnet 5.5 reads Haiku 5.5 and Sonnet 5; nobody reads Fable 5.2's or Sonnet 5.5's. Unreadable blocks are silently dropped, so carry key reasoning in a summary when moving down or off Fable.
- **Subagents on smaller models** for work where mistakes are cheap to spot (find files, run tests, read logs). Keep edits and judgment on the main model: a misread search sends the main model down a costly detour.

  ```markdown
  ---
  name: log-reader
  description: Reads test output and logs and reports each failing case. Use after a test run.
  tools: Read, Grep, Glob, Bash
  model: haiku
  ---
  For each failure, report the file, test name and first error line. Do not edit files.
  ```
- **Explore** runs on the session model, not Haiku; define your own `Explore` agent with `model: haiku` to keep it cheap. The `haiku` alias is still Haiku 4.5 on Bedrock/Google Cloud/Foundry; pin with `ANTHROPIC_DEFAULT_HAIKU_MODEL`.
- **`opusplan`** plans on Opus and edits on Sonnet, the opposite of "edits on the main model"; measure before defaulting to it.
- **Advisor tool (beta)**: cheap executor consults a stronger advisor. Results modest: Opus 5.5 `high` + Fable 5.1 advisor gained 1.7 pts for ~2.1× cost, within noise. Pairings are restricted (Sonnet 5.5 executor rejects Opus 4.8/4.7 and Sonnet 5; Fable 5.2 executor accepts only Fable/Mythos 5.2).
- **Lead + workers** for work too big for one context: Opus 5.5 lead over Sonnet 5.5 workers matched Opus alone at ~1/3 less cost. Skip for dependent chains or work that fits one context.

## Subagent model selection (Claude Code)

`model` takes an alias, full ID, or `inherit`. Precedence:

1. Per-invocation `model` on the Agent call (aliases only).
2. The definition's `model` frontmatter.
3. `CLAUDE_CODE_SUBAGENT_MODEL`.
4. The main conversation's model.

A family alias in 1–2 resolves to the main model's exact version if it's the same family. Add `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` to force the env model on every subagent and ignore definitions. The env var doesn't change built-in Explore/Plan. A subagent on a different model doesn't inherit session effort; set `effort` in its definition. Check with `/tasks`.

## Sonnet 5 → 5.5 migration

Swap the ID to `claude-sonnet-5-5`, then (`/claude-api migrate this project to claude-sonnet-5-5` does this in Claude Code):

1. `thinking: disabled` → 400; use `{"type": "between_tools"}` (only at low/medium/high, no other fields, effort fixed for the conversation; adaptive otherwise).
2. Forced `tool_choice` (`any`/`tool`) → 400; use `auto` + `strict: true` tool + prompt instruction.
3. Keep history append-only; thinking blocks are bound to model and conversation.
4. Computer use: `computer_toolset_20260801` only (API, Google Cloud); drop the old beta and fine-grained-streaming headers, set per-tool `eager_input_streaming`.
5. Advisor: Opus 4.8, Opus 4.7, Sonnet 5 rejected as advisors.
6. Interim text between tool calls now arrives as thinking blocks; set `thinking.display` to `"updates"`/`"summarized"` to show it, and pass blocks back unchanged.

```python
# Before: thinking={"type": "disabled"}, output_config={"effort": "xhigh"}
# After:  thinking={"type": "between_tools"}, output_config={"effort": "high"}
```


## Compatibility checklist

- **Read content blocks by type**; a thinking block may come first, so `content[0].text` breaks.
- Disabled thinking, forced `tool_choice` and `computer_20251124` all → 400 on Sonnet 5.5, Opus 5.5, Fable 5.2 (Haiku 5.5 can disable thinking at `high` or below).
- **Fable 5.2 preserved thinking**: editing anything before a Fable thinking block → 400; keep history append-only (Claude Code/Agent SDK handle it).
- **Opus 5 → 5.5**: default effort dropped `high` → `medium`; re-sweep.
- **Haiku 4.5 → 5.5**: `budget_tokens` → effort; drop non-default `temperature`/`top_p`/`top_k`; no trailing assistant prefill.
- **Safety fallbacks**: flagged cyber/bio requests may re-run on an older model.
- **Stale IDs**: grep for hard-coded `claude-` IDs and compare with the facts table.

## Apps that host an agent (Agent SDK, Claude Code, ACP)

- **Use the agent's default model** (Opus 5.5), not a pinned ID; let users override. SDK `setModel("default")`; `ANTHROPIC_MODEL` sets the start.
- **Accept aliases** (`opus`, `sonnet`, `haiku`, `fable`); the agent resolves them.
- **Show refused models**: if `setModel()` or an ACP adapter rejects an ID, show the error and fall back to default.
- **Expose effort**: SDK `effort` / `applyFlagSettings({ effortLevel })` (`null` = model default); CLI `--effort`/`/effort`/`CLAUDE_CODE_EFFORT_LEVEL`; ACP: offer only levels the model advertises; raw API `output_config.effort`.
- **Don't pin effort by accident**: an explicit level overrides user settings and survives model switches; omit it to follow the user's. Never let a global default overwrite a per-session or per-model choice, or another provider's default effort leak into Claude turns.
- **Send model, effort, fast mode only when they change**: changes can break the cache.
- **Fast mode** (Opus 5.5, Fable 5.2) via `fastMode` setting or ACP option; it keeps a separate cache, so set it at session start.
- **Subagent models**: put `CLAUDE_CODE_SUBAGENT_MODEL` (and `_FORCE=1`) in the agent process's environment, or set `model:` per definition; read/search subagents on Haiku or Sonnet 5.5.

## Measure before committing

1. Sample real tasks, including the hardest; write an outcome check for each.
2. Run each candidate model at 2–3 effort levels in separate sessions; record cost per completed task from `usage`.
3. Chart score vs spend; a multi-model setup stays only if it beats every single-model point.
4. Shadow-run the winner on real traffic before cutting over.

Quick version: run 3–4 real tasks on two models via `/model`; in `/usage` check:

1. **Cache share** is high on long sessions; if not, look for a pause, model switch, mid-session MCP connect, or effort change on a cloud provider.
2. **Output vs input**: heavy output on a small change means effort too high or retries.
3. **Total input vs conversation size**: many multiples means many turns; find the loop.

## How to answer the user

1. Name the model and effort, with a one-line reason.
2. Say what would make them move up or down; remind them verification and effort come before switching model.
3. If integrating via API, mixing models, or migrating, flag relevant compatibility items.
4. If cost matters, give the relevant numbers and suggest the measurement loop rather than trusting list prices.

## Based on

Credit only; everything needed is above.

- Addy Osmani, "Which Claude model should you build with? Fable, Opus, Sonnet, or Haiku", claude.dev Blog (Anthropic), Nov 9, 2026. https://claude.dev/blog/which-claude-model-should-you-build-with
- Addy Osmani, "Building with Claude Sonnet 5.5", claude.dev Blog (Anthropic), Sep 28, 2026. https://claude.dev/blog/building-with-claude-sonnet-5-5/
- Addy Osmani, "What a task costs on Opus 5.5", claude.dev Blog (Anthropic), Sep 25, 2026. https://claude.dev/blog/what-a-task-costs-on-opus-5-5
- Anthropic, "Effort", Claude Platform docs. https://platform.claude.com/docs/en/build-with-claude/effort
- Anthropic, "Choosing the right model", Claude Platform docs. https://platform.claude.com/docs/en/about-claude/models/choosing-a-model
- Anthropic, "Create custom subagents", Claude Code docs. https://code.claude.com/docs/en/sub-agents
- Anthropic, Agent SDK TypeScript reference, Fast mode, Environment variables, and Prompt caching docs (for the hosted-agent section).

The interactive-coding row and the hosted-agent section combine the articles with these docs; they are this skill's synthesis.
