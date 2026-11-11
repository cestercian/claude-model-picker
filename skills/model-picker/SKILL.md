---
name: model-picker
description: Choose the right Claude model (Haiku 5.5, Sonnet 5.5, Opus 5.5, Fable 5.2) and effort level for a task, subagent, API integration, or an app that hosts an agent through the Agent SDK. Use when the user asks which model to use, wants to cut cost or latency, is configuring subagent models, mixing models in one system, or migrating a request from one Claude model to another.
---

# Claude model picker

Pick a starting model and effort level, then adjust with evidence. Everything needed to decide is written out below; you should not need to fetch anything.

Facts here (prices, IDs, defaults, version numbers, benchmark figures) reflect Anthropic's published guidance as of November 2026. If the user's environment shows something different (an error message, `/effort status`, `/model`, a response's `usage`), trust the environment and say the skill may be out of date.

## Contents

1. Short answer
2. The lineup and model facts
3. Task → starting point
4. Effort: what it is, levels, per-model defaults and advice
5. Effort before model: the tuning order
6. Trade-offs with numbers (capability/cost, latency, context/cost)
7. What a task actually costs (cost anatomy and worked numbers)
8. Mixing models (switching, thinking blocks, subagents, opusplan, advisor, lead+workers)
9. Subagent model selection in Claude Code (full precedence)
10. Sonnet 5.5 in depth (choosing vs Opus, pricing, details, Sonnet 5 → 5.5 migration, tuning, refusals)
11. Compatibility checklist before migrating
12. Apps that host an agent (Agent SDK, Claude Code, ACP)
13. Measure before committing (and reading a `/usage` session)
14. How to answer the user
15. Based on

---

## 1. Short answer

**Start on Opus 5.5 at medium effort** (its default). It is Claude Code's default model on Pro, Max, Ultra, Team, Enterprise and the Anthropic API; it costs half of Fable 5.2 per input and output token; and the platform's own selection guide names it as the starting point for most coding and knowledge work. Move only with a reason:

- **Down to Sonnet 5.5** for well-scoped everyday work: a bug fix, iterating on a feature, building against a spec. A clear spec is *not* by itself a reason to drop down: a migration or a performance optimization can be fully specified and still hard enough for Opus 5.5 (or Fable 5.2 if it is a big one).
- **Further down to Haiku 5.5** for the simplest high-volume work (classification, routing, tagging, extraction) and for subagents doing side tasks for a bigger model.
- **Up to Fable 5.2** for the hardest technical work in coding, finance and the sciences; major refactors and migrations; architecture; leading many subagents; long unattended runs; or when Opus 5.5 at `high` still falls short.

**Try effort before switching model.** Effort is the finer dial. Past `high`, one more level can cost as much as the next model up, so at that point test the bigger model at a lower effort instead. Going one level *down* can also hold quality at lower cost. More effort buys more thinking, more tool calls and more self-checking, and it adds latency; it does not add knowledge or skill the model lacks. For that you need a bigger model.

---

## 2. The lineup and model facts

Generally available: Claude Haiku 5.5, Claude Sonnet 5.5, Claude Opus 5.5, Claude Fable 5.2. Claude Mythos 5.2 has Fable 5.2's capabilities but is only open to organizations admitted through Anthropic's verification programs (for example the Cyber Verification Program); it is the most capable option for cybersecurity and life-sciences research.

| | Haiku 5.5 | Sonnet 5.5 | Opus 5.5 | Fable 5.2 |
|---|---|---|---|---|
| API model ID | `claude-haiku-5-5` | `claude-sonnet-5-5` (Bedrock: `anthropic.claude-sonnet-5-5`) | `claude-opus-5-5` | `claude-fable-5-2` (Mythos: `claude-mythos-5-2`) |
| Claude Code alias | `haiku` | `sonnet` | `opus` | `fable` |
| Built for | high-volume, cost-sensitive work | fast, capable help on well-scoped tasks | daily driver for complex coding and knowledge work | most ambitious, open-ended technical work |
| Relative latency | fastest | fast | moderate | moderate |
| Input / output per MTok | $0.10 / $0.50 (≤100K prompt); $0.50 / $2.50 (>100K) | $2 / $10 | $4 / $20 | $8 / $40 |
| Cache read per MTok | $0.01 (≤100K); $0.05 (>100K) | $0.10 | $0.20 | $0.20 |
| Context window | 1M | 1M | 1M | 1M |
| Max output | 128K | 128K (300K on Message Batches with beta header `output-300k-2026-03-24`) | 128K | 128K |
| Thinking on the API | adaptive, on by default; can be turned off at `high` effort or below | adaptive, on by default; `between_tools` skips upfront thinking; cannot be disabled | adaptive, always on; disabling = error | adaptive, always on; disabling = error |
| API default effort | `medium` | `high` (`medium` in Claude Code) | `medium` | `high` (`medium` in Claude Code) |
| Effort levels | low, medium, high, xhigh, max | all five | all five | all five |
| Fast mode | no | no | yes | yes |


Other price lines worth knowing:

- **Cache writes** are 1.25× the input price for a 5-minute cache and 2× for a 1-hour cache. Opus 5.5: $5 (5-min) / $8 (1-hour). Sonnet 5.5: $2.50 / $4.
- **Fast mode** (Opus 5.5, Fable 5.2 only): up to 2.5× output speed at premium pricing. On Opus 5.5 that is 2× standard: $8 input / $40 output per MTok. On a Claude subscription it bills to usage credits, not plan limits.
- **Batch API**: half price on input and output.
- **US-only inference** (`inference_geo: "us"`): 1.1× standard price.
- Prices are Claude API list prices. On Pro/Max/Team plans the dollar figure is a guide to work done, not a bill.

Ratios to reason with:

- Opus 5.5 cache read = 1/20 of its input price; an Opus 5.5 output token costs 100× a cache read.
- Fable 5.2 vs Opus 5.5: 2× on fresh input and output, *same* $0.20 cache read.
- Sonnet 5.5 is half Opus 5.5 on input and output.
- Haiku 5.5's short tier is 1/20 of Sonnet's input price; even its long tier (>100K) is 1/4 of Sonnet.

Speed changes generation to generation: Fable 5.2 produces output up to 30% faster than Fable 5.1, and Sonnet 5.5 is 30% faster than Sonnet 5.

---

## 3. Task → starting point

| Task | Start with | Starting effort | Why |
|---|---|---|---|
| Everyday engineering and debugging; multi-PR feature work | Opus 5.5 | medium (default), then high | Best balance of cost and capability for daily work |
| Classification, routing, tagging, high-volume extraction | Haiku 5.5 | medium (default) | Cheapest and lowest-latency model in the family |
| Subagents that search files, read logs, summarize | Haiku 5.5 or Sonnet 5.5 | the model's default | Mistakes are cheap to catch, and judgment calls stay with the main model |
| Chat and interactive sessions | Sonnet 5.5 (Haiku 5.5 for simple chat at volume) | low or medium | Sonnet is fast, especially at low/medium |
| Interactive coding (a person waits while the agent edits code) | Opus 5.5 by default; Sonnet 5.5 when edits are scoped and waiting is the main cost | medium | This skill's synthesis: keep edits on the strong model unless latency dominates |
| Well-scoped coding: bug fixes, feature iteration, spec-driven work | Sonnet 5.5 | medium, then high for harder tasks | Handles scoped work well at half the Opus token price |
| Everyday documents, slides, spreadsheets | Sonnet 5.5 | high (API default; medium in Claude Code) | Iterates fast with a good eye for design. Start on Opus 5.5 for a complex financial model, or for a document assembled from a big stack of sources |
| Repeated agent tasks: investigation, review, drafting | Sonnet 5.5 | medium | A bigger model is rarely needed for well-defined tasks that repeat |
| Computer use, vision-heavy workflows | Opus 5.5 | medium | Long agentic loops over many screenshots: what Opus 5.5 is built for |
| Major refactors and migrations, architecture planning, leading many subagents | Fable 5.2 | medium, then high (API default is high) | Most capable GA model for technical work |
| Deep research; scientific or quantitative analysis | Fable 5.2 | medium, then high (API default is high) | Built for the hardest technical projects |
| Not sure yet | Opus 5.5 | medium (default) | Default in Claude Code and the recommended starting point |

The official selection matrix, same idea in different words:

| You need | Start with | Examples |
|---|---|---|
| Highest available capability | Fable 5.2 | hours-long agent sessions, multistep deep research, analysis carried to a finished doc/spreadsheet/deck |
| Complex agentic coding and enterprise work | Opus 5.5 | multi-hour autonomous coding, large refactors, complex systems engineering, vision-heavy work, computer use |
| Speed plus capability for everyday coding/agent/enterprise work | Sonnet 5.5 | code generation, data analysis, content creation, visual understanding, agentic tool use |
| Lowest latency and price | Haiku 5.5 | real-time apps, high-volume processing, cost-sensitive deployments that still need reasoning, subagent tasks |

### Two ways to approach a new application

- **Efficiency-first** (prototypes, tight latency, cost-sensitive, high-volume straightforward work): build on Haiku 5.5, test thoroughly, check whether it meets the bar, and upgrade only for the specific gaps you find.
- **Capability-first** (complex reasoning, science/math, nuanced understanding, accuracy over cost, advanced coding, high-autonomy agents): build on Opus 5.5, tune prompts for it, evaluate, then claw back cost by lowering effort or moving parts of the workflow to smaller models. If evals at `xhigh` or `max` still miss on demanding reasoning or long-horizon agent work, move to Fable 5.2.

Criteria to weigh in either case: required capabilities, how fast replies must come back (fast mode is an option on Opus 5.5/Fable 5.2), budget for development and production, and effort, which is often a better lever than switching models.

---

## 4. Effort

### What effort controls

Effort is a behavioral signal for how many tokens the model spends, not a hard budget. It affects **every output token**: response text, tool calls and their arguments, and thinking. So it works with or without thinking.

- Lower effort: fewer and terser tool calls, multiple operations combined into one call, straight to action without preamble, short confirmations. At low levels the model may skip thinking entirely on simple problems, though it still thinks on genuinely hard ones.
- Higher effort: more tool calls, a stated plan before acting, fuller summaries, more code comments; thinks more readily and longer.
- In a tool loop, follow-up requests that only process tool results may skip thinking at any level.
- `thinking` decides whether the model thinks in thinking blocks; `effort` decides how much work goes into the whole response. `adaptive` is a thinking mode, never an effort value.
- Effort barely changes visible reply length on several models. To change length or tone, prompt for it.
- Asking the model in the system prompt to "think less" does not reliably reduce thinking. Lower the effort level instead.

### Setting it

- Messages API: `output_config: {"effort": "medium"}`. Omitted means the model's default. Sending the default value is identical to omitting it.
- Managed Agents: `effort` on the agent's model object, or in a session's model override.
- Claude Code: `/effort <level>`, `/effort status`, `--effort`, or `CLAUDE_CODE_EFFORT_LEVEL` (which overrides the others). Changing it mid-task applies from the next request.
- Agent SDK: the `effort` option of `query()`, or `applyFlagSettings({ effortLevel })` mid-session.

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=4096,
    messages=[{"role": "user", "content": "..."}],
    output_config={"effort": "medium"},
)
for block in response.content:   # read by type: a thinking block may come first
    if block.type == "text":
        print(block.text)
```

### Levels

| Level | Meaning | Typical use |
|---|---|---|
| `low` | most efficient; big token savings, some capability loss | simple, speed/cost-sensitive work; subagents; mechanical edits (renames, applying a known pattern across files) |
| `medium` | balanced, moderate savings; default on Opus 5.5 and Haiku 5.5 | agentic tasks balancing speed, cost and quality; well-scoped daily work |
| `high` | spends what the task needs; default on most other models | complex reasoning, hard coding, agentic tasks; the step to take when medium stalls |
| `xhigh` | extended capability for long-horizon work | agentic/coding runs over ~30 minutes with token budgets in the millions |
| `max` | no limit on spend | deepest reasoning; measuring a model's ceiling; rare tasks where time and tokens don't matter |

Not every model with `max` has `xhigh`. Among current models all four (Haiku 5.5, Sonnet 5.5, Opus 5.5, Fable 5.2) support all five levels. Levels are calibrated per model: the same name does not mean the same amount of thinking across models or generations. Never carry a level over from an older model; re-sweep.

### Per-model defaults and advice

- **Fable 5.2** (and Mythos 5.2): default `high`; thinking always on, so effort is the main control. Most of the capability gain is already there at `high`; above that gains are uneven and task-dependent. Use `xhigh` for the hardest long-running coding/agent work where you've measured a gain; drop to `medium`/`low` where evals hold. Keep `max` for ceiling measurements: it runs slower and costs more than `xhigh`, often for a small gain. Set a large `max_tokens` at `high` and above (it caps thinking plus reply). Re-sweep rather than reusing Fable 5.1 settings.
- **Opus 5.5**: default `medium` (Opus 5 and earlier defaulted to `high`, so an omitted effort now runs one level lower than before). Thinking always on; `thinking: {"type": "disabled"}` returns 400 at every level. Opus 5.5 thinks more per turn than Opus 5 at the same level, especially at `xhigh`/`max`, so do not reuse your Opus 5 level. Start at `medium`; keep `xhigh`/`max` for measured gains; set a large `max_tokens` at higher levels. (Claude Code describes Opus 5.5 as having low/medium/high/xhigh plus `max` for a single session.)
- **Sonnet 5.5**: default `high` on the API, `medium` in Claude Code. Levels recalibrated versus Sonnet 5. Start at `high` unless agentic or latency-sensitive. Agentic coding and multistep tool use: `medium` for well-specified tasks, `high` for harder/longer ones. Chat and latency-sensitive: `medium` or `low`. `xhigh`/`max` only where evals show gains; at those levels Sonnet thinks long enough to lose its speed/cost edge, so compare against Opus 5.5. Thinking counts toward `max_tokens`; for agentic coding set `max_tokens` to 128,000 and stream. Lowest thinking setting is `between_tools` (works at low/medium/high only).
- **Haiku 5.5**: default `medium`; start there for most work, including agentic coding. `low` for chat, short tool tasks, simple high-volume requests; but in long agent prompts `low` makes it more likely to skip a search, stop early or skip a check. `high` for knowledge work, longer agent tasks, strict instruction following. `xhigh`/`max` only with measured gains, and compare against Sonnet 5.5 on quality, cost and speed. Can send `thinking: {"type": "disabled"}` at `high` or below (400 at `xhigh`/`max`).
- **Older models, briefly**: Fable 5.1 and Fable 5 default `high`. Opus 5 default `high`; step to `xhigh` for demanding agent work; set `max_tokens` around 64K at `xhigh`/`max`; cannot disable thinking at `xhigh`/`max`. Opus 4.8 and 4.7: start at `xhigh` for coding and agent work, `high` for other intelligence-sensitive work. Sonnet 5 default `high`. Sonnet 4.6 defaults to `high` but `medium` is the recommended setting; set it explicitly to avoid surprise latency.

### Changing effort mid-conversation (and the cache)

- **Top-level change**: setting a different `output_config.effort` on the next request changes the rendered prompt, so the cached prefix is lost and the next turn pays cache writes. If the model lacks per-message effort, pick a level at the start and hold it.
- **Per-message effort (beta)**: keeps the cache. Supported on Fable 5.2, Mythos 5.2, Fable 5.1, Mythos 5.1, Opus 5.5, Opus 5, Sonnet 5.5 and Haiku 5.5 (platform coverage varies: on Amazon Bedrock only Fable 5.2, Mythos 5.2, Fable 5.1, Mythos 5.1 and Opus 5.5; Opus 5 does not have it on Bedrock). Requires beta header `mid-conversation-output-config-2026-07-01`. Insert a system message with empty content carrying the new level:

  ```json
  {"role": "system", "content": [], "output_config": {"effort": "low"}}
  ```

  Placed right after a user turn with new input, it applies to the reply to that turn. Placed elsewhere (for example between an assistant turn and the next user turn), it applies from the next user turn with new input. A user turn holding only `tool_result` blocks does not count as new input. The level holds until changed again. It can sit anywhere in `messages`, including first.
- Errors: without the beta header you get `messages.N.output_config: Extra inputs are not permitted`. On a model without per-message effort (e.g. Fable 5) you get `output_config.effort requires a model that supports per-turn effort; this model does not` (Bedrock returns the "Extra inputs" error instead).
- Locks: Sonnet 5.5 with `between_tools` and Haiku 5.5 with thinking disabled cannot change effort mid-conversation (a differing per-message effort returns 400). Use adaptive thinking if you need to vary it.
- Prefer per-message over top-level changes: besides the cache, some models (Fable 5.1 notably) follow a top-level change less reliably because their earlier replies were written at the old level.
- **In Claude Code**: on Opus 5.5 with an API key or a Claude subscription, `/effort` changes keep the cache, so you can raise effort for one hard step and drop it again. On Amazon Bedrock, Google Cloud's Agent Platform, or a Claude apps gateway, an effort change still clears the cached conversation; change it at a natural break there.

Best-practice summary: set effort explicitly; use `low` for speed-sensitive or simple tasks; test per use case; vary effort by task complexity across workloads rather than within a cached conversation (or use per-message effort).

---

## 5. Effort before model: the tuning order

When results fall short, in this order:

1. **Give the model a way to verify its own work**: a test, a build, a type-check, a script that calls the endpoint. A check costs one turn and its output; extra effort adds thinking to *every* turn. Usually cheaper and lower-latency than raising effort.
2. **Raise effort one level.**
3. **Move up one model**, and start it at a *lower* effort than you used on the smaller one.

Why the order works, illustrated: an API field is renamed. At `medium` the model fixes the handler and its tests pass, but the client keeps sending the old field name. It did what was asked but didn't read far enough to find the second caller. At `high` it reads more call sites first and fixes both layers in one pass. But a test that exercises the client would have caught the bug at `medium` on the turn it was written. So check for a verification path before paying for more effort.

When to escalate to the bigger model: don't wait for a third failure. If Opus 5.5 at `xhigh` hits the same problem twice, switch up, and switch back once it is solved. For interactive work Opus 5.5 is still the better fit (lower latency, lower price).

Rough break-even for effort (Opus 5.5 prices): if `high` adds ~20K thinking tokens across a task, that is ~$0.40. A ten-turn retry loop at 100K cached context with 10K total output costs about the same. So `high` pays for itself when it saves one retry, and is wasted on tasks `medium` would have finished first time.

**Low then re-run failures**: if outcomes are machine-checkable, run everything at a low level and re-run only failures at a high level. On a SWE-bench Pro subset, running Opus 5.5 at `low` and re-running the failures at `high` passed ~97% of tasks at ~$0.17 each, versus 95.3% at $0.29 each with every task at `high`. Better on both axes.

**Need speed without losing capability**: use fast mode on Opus 5.5 or Fable 5.2 (premium price) instead of dropping effort or model.

---

## 6. Trade-offs with numbers

### Capability vs cost per finished task

Bigger models cost more per token and are generally more capable, but per-token price does not predict per-task cost or quality.

- **Terminal-Bench 3.0** (Anthropic's runs): Fable 5.1 at `max` scored 58.0%; Opus 5.5 at `high` scored 58.9% using half the tokens.
- **FrontierCode 2.0**: Fable 5.2 at `high` ≈ Opus 5.5 at `max` (45.3% vs 45.5%) at roughly half the cost per task. Rule: before pushing Opus 5.5 past `high` on a hard task, test Fable 5.2 at `high`.
- Small models do well on high-volume work with checkable outputs; they are weaker at long agentic loops.
- **Price the hardest tasks, not just the typical ones.** On one 20-problem search benchmark, two problems accounted for 43% of total spend. A comparison that only samples typical tasks will mislead.

### Latency vs accuracy

A faster setup finishes sooner and misses more. Decide how long a person can wait first. For replies someone waits on, start Sonnet 5.5 at `low` or `medium`.

Effort roughly tracks latency and cost. SWE-bench Pro, Opus 5.5, relative to `high`:

| Effort | Score vs high | Cost vs high |
|---|---|---|
| `low` | ~8 points lower | ~1/3 |
| `medium` | ~2.5 points lower | ~70% |
| `high` | baseline | 1× |
| `xhigh` | ~1.4 points higher | ~2.5× |

### Context vs cost

All current models have a 1M-token window. A large context helps but is resent on every turn; on long runs most of that resend bills as cache reads.

- Cache read per turn on Opus 5.5: ~$0.004 at 20K context, ~$0.03 at 150K. Thirty turns at 150K ≈ $0.90 in reads alone, versus ~$0.12 at 20K. On Claude 4.6 and later, a bigger window does not raise the per-token price; the cost is purely the resend.
- Fable 5.2 and Opus 5.5 share the $0.20 cache-read rate, so on long cache-heavy runs their cost gap is only in uncached input and output (where Fable is 2×). Sonnet 5.5's cache read is half Opus's (see the pricing footnote).
- Therefore: **the bigger the main model, the more a lean main context matters.** Keep the expensive model's context for decisions; push verbose reading to cheaper subagents.
- **Haiku 5.5 is the only model priced by prompt length**: ≤100K tokens bills at the low tier; >100K bills at 5× (still 1/4 of Sonnet). It is cheapest on short prompts (1/20 of Sonnet's input price). When every call needs a long prompt, compare Haiku vs Sonnet on cost per finished task, not list price.

---

## 7. What a task actually costs

A task is a loop of requests: read the conversation, call a tool, read the result, repeat. Four things set the bill:

1. **Turns.** Every turn resends the conversation so far. The cheapest turn is the one you don't need. Verification paths, gathering context in one pass and batching tool calls all cut turns.
2. **Cache reads.** Most of each resend was seen last turn and bills at the cache-read price. Hit rate is the single biggest input-cost lever.
3. **Output tokens**, including thinking (billed even when only a summary is shown). Output is 5× input price; this is why effort moves the bill so much.
4. **Model**, which sets the price of every token, including every subagent that inherits it.

Worked numbers at Opus 5.5 list prices ($4 in, $20 out, $0.20 cache read; cache writes ignored; illustrative token counts):

- **Turns**: context grows 20K → 120K over 40 turns; average turn ~70K; ~2.8M input tokens total though the conversation never exceeded 120K. At 90% cached, input ≈ $1.62. The same task in 25 turns ≈ 1.75M tokens ≈ $1.02.
- **Cache hit rate** on those 2.8M tokens: 0% cached = $11.20; 90% = $1.62; 96% ≈ $0.99. Nothing else moves input cost this much.
- **Output**: 60K output tokens = $1.20, the same as reading 6M tokens from cache.
- **Model effect**: cheaper cache reads mainly help long sessions; cheaper output mainly helps reasoning-heavy tasks.

### Opus 5.5 vs Opus 5

- Input and output are 20% cheaper than Opus 5; cache reads 60% cheaper (from 1/10 of input price to 1/20). On subscriptions, the lower price stretches limits ~25% further; five-hour limits were also raised on Pro, Max, Team and seat-based Enterprise.
- Same-token session priced both ways: 2.0M cache reads ($1.00 → $0.40), 200K fresh input, 60K output; total $3.50 on Opus 5 vs ~$2.40 on Opus 5.5, about 31% less from price alone. At 10 tasks/day over 22 working days: ~$770 vs ~$528 a month.
- Mostly-cached sessions save up to 60% on input; short uncached questions with long answers save up to ~20% (output dominates).
- The headline "~40% cheaper to run" is an estimate for typical token-billed workloads at default settings, combining lower prices with fewer tokens per task at the `medium` default. It is not a 40% per-token cut. Opus 5.5 can also use *more* tokens on some answers since it always thinks. Gains are smallest on well-scoped tasks (similar turn counts) and largest on open-ended ones where an older model wanders.
- Long runs end with a report of what changed, what was found and what's needed from you, which reduces reruns.

### Habits that cut cost (Claude Code)

- **Raise effort before changing models**; use `low` for mechanical edits, `medium` for well-scoped work, `high` when medium stalls.
- **Model roles**: small model for lookups, Opus 5.5 for supervised work (features across a few files, debugging, review with follow-up edits), bigger model for the hardest work (unsupervised long runs, problems with no existing pattern, big changes coordinating many subagents). For a mechanical edit across many files, keep Opus 5.5 at `low` rather than moving the edit to a smaller model.
- **Older reference point**: Fable 5.1 listed at $10/$50 with $0.25 cache reads (0.025× its input). The current Fable 5.2 is $8/$40 with $0.20 cache reads.
- **Agent teams** (experimental) multiply cost: each teammate is a separate Claude Code instance burning tokens until it exits; with teammates in plan mode, expect about 7× the tokens of a standard session. Keep teams small and tasks self-contained, and shut teammates down when done.
- **Prompt audit on migration**: instructions written for older models (mandatory multi-step rituals, scratchpad rules, "verify twice", contradictory rules) make newer models write more and repeat tool calls. `/claude-api prompt-audit` checks skills, CLAUDE.md and app code. On one 44-ticket support benchmark, moving Opus 4.8 → Opus 5.5 at `low` cut cost ~18%, and the audit cut a further ~9% (~25% total). One benchmark; treat as an example.
- **Cache mechanics**: cached read on Opus 5.5 = 5% of fresh input; 5-minute write = 1.25× input, 1-hour = 2×; each hit resets lifetime for free. In Claude Code the lifetime is 1 hour on a subscription, 5 minutes on an API key or cloud provider (and on a subscription once it draws usage credits). At 120K context on Opus 5.5: 5-minute write ≈ $0.60, read ≈ $0.02 (one write = 25 reads), so a six-minute break on an API key turns a $0.02 read into a $0.60 write; a 1-hour write ≈ $0.96.
- **Things that force a cache write**: a pause longer than the lifetime; changing effort on Bedrock / Google Cloud Agent Platform / a gateway; turning fast mode on for the first time in a conversation (the first fast request pays fast-mode input on the whole conversation, uncached); connecting/disconnecting an MCP server; switching models; compaction. Changing tool definitions clears the whole cache; changing the system prompt clears from that point (nearly everything). Set these up at session start and leave them alone.
- **`/clear` vs `/compact`**: `/clear` is free; use it between unrelated tasks. `/compact` costs one request; at 150K tokens ≈ $0.25 (read + a few-thousand-token summary + new cache write) and saves ≈ $0.025 per later turn, so it pays back in ~10 turns. Compact *before* a break while the cache is warm; after a break past the lifetime the same compaction costs ≈ $0.75 in input at 150K. Pass instructions on what to keep (`/compact keep the failing test names and the schema change`), because a summary can drop the one detail that mattered. To abandon a dead end, `/rewind` to a cached earlier turn instead. `/autocompact <tokens>` sets the auto-compaction threshold.
- **What loads before you type**: CLAUDE.md is resent every turn; keep it under ~200 lines. MCP tool definitions are deferred (only names and server instructions load up front); disconnect unused servers via `/mcp`.
- **Typical spend benchmark**: ~$13 per developer per active day across enterprise deployments; under $30/day for 90% of users. A session well above your own norm is worth reviewing.

---

## 8. Mixing models

### Switch at a natural break

A prompt cache is tied to the model that wrote it, so the first turn on a new model pays the cache-write price for the whole conversation. To make that turn smaller: in Claude Code, run `/compact` first or begin a fresh session from a short written plan; on the API, start the new model's conversation from a summary rather than the full history. In Claude Code, `/model` also stores your pick as the default for new sessions, so switch back once the hard part is finished.

### Thinking blocks don't travel down-tier

Each thinking block records which model produced it, and models read only some others' blocks (Claude API and Claude Models on Google Cloud, Google managed):

| Reader | Can read thinking blocks from |
|---|---|
| Fable 5.2 | Opus 5.5, Sonnet 5.5, Haiku 5.5 |
| Fable 5.1 (Claude API) | Opus 5.5 |
| Opus 5.5 | Sonnet 5.5, Haiku 5.5 |
| Sonnet 5.5 | Haiku 5.5 (and Sonnet 5) |
| Haiku 5.5 | none listed |

No model reads blocks from a model above it, and no other model reads Sonnet 5.5's blocks or Fable 5.2's. When the new model can't read earlier blocks, the API silently drops them and the request still succeeds; the new model just works without that reasoning. So moving down a tier, or moving off Fable 5.2 to anything, loses prior reasoning. Plan the switch so a summary carries what matters.

### Subagents on smaller models

A subagent runs in its own context window and returns a summary, keeping its file reads out of the main conversation. It still pays for its own tokens at its own model's rate.

- Put small models on work where a mistake is cheap to spot: finding files, running tests, reading logs, "where is this defined?". Keep code edits and judgment calls on the main model. Reason: if a small model misreads a search result, the main model ends up opening the wrong file and paying for the detour.
- In Claude Code, set `model: haiku` or `model: sonnet` in the agent file; `CLAUDE_CODE_SUBAGENT_MODEL` sets the default for agents without a `model`. Full precedence is in section 9.

Example cheap subagent, saved as `.claude/agents/log-reader.md`:

```markdown
---
name: log-reader
description: Reads test output and logs and reports each failing case. Use after a test run.
tools: Read, Grep, Glob, Bash
model: haiku
---

For each failure in the logs you're given, report the file, test name and first error line. Do not edit files.
```

**Explore gotcha**: in current Claude Code the built-in Explore subagent runs on the session's model instead of always on Haiku. If the session runs Fable, Explore runs on the Opus model the `opus` alias resolves to when connected via a Claude subscription, an Anthropic Console account, or an LLM gateway through `ANTHROPIC_BASE_URL`; on Bedrock, Google Cloud's Agent Platform, Microsoft Foundry, Claude Platform on AWS, or a Claude apps gateway it stays on the session model. To keep exploration cheap, define your own user or project subagent named `Explore` with `model: haiku`; it overrides the built-in and keeps its own model. Plan (used in plan mode) inherits the session model.

**`haiku` alias**: on the Anthropic API it resolves to Haiku 5.5 (needs Claude Code v2.1.293+). It still resolves to Haiku 4.5 on Amazon Bedrock, Google Cloud, Microsoft Foundry and Claude Platform on AWS. Pin what the alias means with `ANTHROPIC_DEFAULT_HAIKU_MODEL`. (Likewise `ANTHROPIC_DEFAULT_OPUS_MODEL` pins `opus`.) The `sonnet` alias resolves to Sonnet 5.5 on the Claude API from Claude Code v2.1.284 (Agent SDK for TypeScript v0.3.284+).

### `opusplan`

A Claude Code alias that plans with Opus in plan mode, then executes with Sonnet. That puts code edits on the cheaper model, the opposite of the advice above (edits on the main model, cheap models only where errors are easy to spot). Measure it on your own tasks before making it a default.

### Advisor tool (beta)

A fast, cheap **executor** runs the task and can consult a stronger **advisor** partway through. Most tokens bill at the executor's rate.

- It helps only if the advisor adds capability the executor lacks *and* the executor actually asks, which it does less at low effort. Measure the executor's consult rate first.
- Results have been modest: of the four pairings in the platform's cost guide, none clearly beat the executor alone. Example: Opus 5.5 executor at `high` + Fable 5.1 advisor on an internal coding benchmark scored 90.1% at $2.92 per attempt, 1.7 points over Opus 5.5 alone at `high` for ~2.1× the cost. That's within run-to-run noise and about what raising effort would buy.
- Most plausible fit: agentic work where most turns are routine.
- Allowed pairings are restricted. With a Sonnet 5.5 executor, Fable 5.2, Opus 5.5, Opus 5 and Sonnet 5.5 are allowed as advisors, while Opus 4.8, Opus 4.7 and Sonnet 5 are rejected. With a Fable 5.2 executor, the only allowed advisors are Fable 5.2 and Mythos 5.2. Advice comes back encrypted as an `advisor_redacted_result` block; your code cannot read it.

### Lead + workers (delegating bulk work)

Different from subagents: this is for work too big for one context window. A top-tier lead splits the job, fans pieces out to many cheaper workers in parallel, and merges results.

- Measured: on an internal defect-search benchmark over ~22M tokens of source, an Opus 5.5 lead over Sonnet 5.5 workers (up to 25 concurrent) scored about the same as Opus 5.5 alone at its default effort while costing roughly a third less.
- On the Claude Platform, Claude Managed Agents' multiagent orchestration provides a coordinator plus worker roster.
- Skip it for a single chain of dependent steps, or work that fits in one context: planning and merging overhead exceeds the savings there.

### Building a multi-model system

Build every request for the **strictest** model so any step can move tiers without code changes: append-only history, `tool_choice: auto` with strict tools, read content blocks by type, and prompts/tool descriptions precise enough for the most literal model. When routing is simple (one model per step type), route in code, not with a model.

---

## 9. Subagent model selection in Claude Code

The `model` field in a subagent definition accepts an alias (`haiku`, `sonnet`, `opus`, `fable`), a full ID (e.g. `claude-opus-5-5`; same values as `--model`), or `inherit` (main conversation's model).

Resolution order (Claude Code v2.1.251+):

1. The per-invocation `model` parameter Claude passes on the Agent call (aliases only; no full IDs or `inherit` there). An installed mod's `agent.spawn` hook model replaces this.
2. The definition's `model` frontmatter (`inherit` = main model).
3. `CLAUDE_CODE_SUBAGENT_MODEL` (alias or ID).
4. The main conversation's model.

Before v2.1.251 the env var came first and overrode everything, including `model: inherit`.

Details that bite:

- A family alias in the parameter or frontmatter resolves to the main conversation's *exact* model if the main model is in that family (so a `sonnet` subagent under a Sonnet session gets the same model and any `[1m]` context suffix). An alias in `CLAUDE_CODE_SUBAGENT_MODEL` always resolves to the alias's target version. Setting the var to `inherit` = unset.
- `CLAUDE_CODE_SUBAGENT_MODEL` alone does not change built-in Explore or Plan.
- **Force one model everywhere** (v2.1.257+): also set `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1`. Then the var's model applies to every subagent, teammate and workflow agent, definition `model` fields are ignored, and Claude can't pass a model. Forks and skills running in a subagent with `model: inherit` still use the main model. With only `_FORCE` set, subagents run on the main model (except built-in Explore follows its own rule).

  ```json
  { "env": { "CLAUDE_CODE_SUBAGENT_MODEL": "haiku", "CLAUDE_CODE_SUBAGENT_MODEL_FORCE": "1" } }
  ```
- An organization `availableModels` allowlist can substitute a blocked value: a blocked family alias falls to the newest allowed version of that family (v2.1.222+); anything else falls back to the inherited model. Interactive sessions show a warning.
- Verify with `/tasks` while a subagent runs (v2.1.242+): it names the model and any per-subagent effort.
- A per-invocation model persists when the subagent is resumed or messaged.
- Subagents inherit the main conversation's thinking on/off setting (v2.1.198+); there's no per-subagent thinking switch.
- **Subagent effort**: set an `effort` field in the definition; Claude can also pass a per-invocation `effort` (v2.1.292+), which overrides the field; `CLAUDE_CODE_EFFORT_LEVEL` overrides both. With neither, a subagent on the main model uses the session's level; a subagent on a different model does *not* inherit the session level, so set `effort` explicitly if it matters.
- Built-in agents with fixed models: `statusline-setup` uses Sonnet; `claude-code-guide` uses Haiku. General-purpose follows the order above.

---

## 10. Sonnet 5.5 in depth

### Sonnet 5.5 vs Opus 5.5

Sonnet 5.5 is the second 5.5-family model, smarter, more token-efficient and 30% faster than Sonnet 5 at the same per-token price; because it needs fewer tokens for the same work, most workloads cost up to ~30% less than on Sonnet 5.

| Workload | Start with |
|---|---|
| Well-scoped everyday coding: bug fixes, feature iteration, checking work against requirements | Sonnet 5.5 |
| High-volume everyday development | Sonnet 5.5 |
| Polished documents, slides, spreadsheets where design sense helps (one-pagers, diagrams, summary slides, edits, spreadsheet cleanup) | Sonnet 5.5 |
| Well-defined agent tasks run repeatedly: investigation, review, drafting | Sonnet 5.5 |
| Complex work needing careful judgment, long-horizon agentic coding and knowledge work | Opus 5.5 |
| The hardest problems | Opus 5.5 (or Fable 5.2) |

Rule of thumb: reach for Sonnet 5.5 when the task comes with a clear spec *and* the result can be checked. For the hardest long-horizon work, use an Opus-class model.

### Pricing and model details

- $2 input / $10 output; cache write $2.50 (5-min) / $4 (1-hour); identical to Sonnet 5 including batch and caching, so swapping the ID doesn't change per-token billing (the total bill changes because fewer tokens are used).
- Surfaces ship different default efforts: `high` on the Claude Platform API, `medium` in Claude Code.
- Images use the high-resolution tier (up to 2576 px long edge); a 2000×1500 image costs ~2.5× the tokens it did on Sonnet 4.6, Sonnet 4.5 or Haiku 4.5. Downscale if you don't need detail.
- IDs: `claude-sonnet-5-5` on Claude API, Claude Platform on AWS, Google Cloud and Microsoft Foundry (Foundry: Global Standard deployments only); `anthropic.claude-sonnet-5-5` on Amazon Bedrock.
- 1M context native (no beta header); 128K max output (300K on Message Batches with `output-300k-2026-03-24`); knowledge cutoff June 2026; same tokenizer as Sonnet 5; minimum cacheable prompt 512 tokens (Sonnet 5: 1,024); rate limits separate from Sonnet 5 with the same default tiers; Priority Tier on the Claude API; zero data retention for eligible customers.
- Sonnet 5.5 also brings per-message effort (beta), system messages mid-conversation, and tool changes mid-conversation (beta).

### Migrating Sonnet 5 → Sonnet 5.5

Change the model ID to `claude-sonnet-5-5`, then handle five breaking changes and one response-shape change. In Claude Code, `/claude-api migrate this project to claude-sonnet-5-5` invokes the bundled Claude API skill and applies the ID swap and parameter changes across the codebase.

**1. Thinking can't be disabled; use `between_tools` to skip upfront thinking.** No `thinking` field means adaptive thinking; `{"type": "disabled"}` returns 400. With `between_tools`, the model thinks only between tool calls, and total response time is the same or faster.

```python
# Before (Sonnet 5)
client.messages.create(model="claude-sonnet-5", max_tokens=16000,
    thinking={"type": "disabled"}, output_config={"effort": "xhigh"}, messages=[...])

# After (Sonnet 5.5)
client.messages.create(model="claude-sonnet-5-5", max_tokens=16000,
    thinking={"type": "between_tools"}, output_config={"effort": "high"}, messages=[...])
```

`between_tools` limits:
- Only at `low`, `medium`, `high`. At `xhigh`/`max` it returns 400; use adaptive thinking there (omit `thinking` or send `{"type": "adaptive"}`). That's why the example drops `xhigh` → `high`.
- Takes no other fields: adding `display`, `budget_tokens` or `block_binding` returns 400.
- Effort can't change mid-conversation under it (differing per-message effort → 400). Use adaptive to vary effort per turn.
- The short progress notes between tool calls still arrive as thinking blocks with summary text. Read blocks by type and pass them back unchanged with the rest of the assistant turn. With no tools, the response is text only.
- Works on every platform offering Sonnet 5.5, no beta header. Update your SDK if it doesn't know `between_tools`.
- For tool-free requests that still need a few steps of working out, use adaptive thinking rather than `between_tools`.

**2. Forced tool choice is gone; use `auto` + strict tool + prompt.** `tool_choice` of type `any` or `tool` returns 400, including on the token-counting endpoint.

```python
weather_tool = {
    "name": "get_weather",
    "description": "Get the current weather in a given location",
    "input_schema": {"type": "object",
                     "properties": {"location": {"type": "string"}},
                     "required": ["location"],
                     "additionalProperties": False},
    "strict": True,
}
client.messages.create(model="claude-sonnet-5-5", max_tokens=1024, tools=[weather_tool],
    tool_choice={"type": "auto"},   # was {"type": "tool", "name": "get_weather"}
    messages=[{"role": "user", "content": "What's the weather in Paris? Use the get_weather tool."}])
```

Strict tools need `additionalProperties: false` on every object.

**3. Keep conversations append-only.** Thinking blocks from Sonnet 5.5 are tied to both the model and the conversation. Sonnet 5.5 reads Sonnet 5's blocks (so switching an existing Sonnet 5 conversation up keeps its reasoning); no other model reads Sonnet 5.5's.

**4. Computer use moves to the toolset.** On the Claude API and Google Cloud, only `{"type": "computer_toolset_20260801"}` works; declaring `computer_20251124` returns 400. Remove the `anthropic-beta: computer-use-2025-11-24` header; in SDKs drop the `betas` parameter and call Messages through the standard client, not the beta namespace. Replace the tools entry and update the agent loop to handle member `tool_use` blocks, batched actions, and `toolset_name` on results. Also remove `fine-grained-tool-streaming-2025-05-14` (with a toolset entry it returns 400) and set `eager_input_streaming: true` on each tool that needs it. Amazon Bedrock still accepts `computer_20251124`.

**5. Check your advisor pairing.** As a Sonnet 5.5 executor it rejects Opus 4.8, Opus 4.7 and Sonnet 5 as advisors; accepts Opus 5.5, Opus 5, Sonnet 5.5 (and Fable 5.2). Advice comes back as an encrypted `advisor_redacted_result` block.

**6. Text between tool calls now arrives in thinking blocks** (no error, but a UI can silently stop showing the model's interim notes). Any note over a sentence or two is returned as a progress-update thinking block, and those blocks show nothing at the default `display`. To see them under adaptive thinking, set `thinking.display` to `"updates"` (beta, header `thinking-display-updates-2026-08-18`) or `"summarized"`, and render each non-empty thinking block before the following `tool_use` block. With `between_tools`, the text arrives without needing `display`.

Coming from Sonnet 4.6 or older, or from Haiku 4.5, there are separate per-model checklists; the Haiku 4.5 items are in section 11.

### Tuning Sonnet 5.5

- **Re-sweep effort**: levels are recalibrated; old settings don't carry over. Defaults per section 4.
- **`max_tokens`**: thinking counts toward it; leave room. For agentic coding use 128,000 and stream.
- **Strip Sonnet 5 workarounds** (refusal steering, tool-call retry shims, "do not be lazy") and re-run evals before any other tuning; existing prompts generally work unchanged.
- **Demand real checks at low effort**: at `low` it sometimes reports a change done without running anything that exercises it. If you see that, add a system-prompt rule along these lines (paraphrase): after changing runnable code, run a real check that exercises the change (tests, type-checker, build, or the changed command) before calling it done; a syntax-only check or a command that failed to start doesn't count; if only declared dependencies are missing, install them via the project's own package manager and lockfile (never sudo or the system package manager) unless told not to; if no real check can run, say which check was skipped and why instead of claiming completion.
- **Don't ask for reasoning in the reply**: that invites `reasoning_extraction` refusals. Use `thinking: {"type": "adaptive", "display": "summarized"}`, or `display: "updates"` (beta) for user-facing progress notes; if you want updates at fixed points (a line before the first tool call, a recap at the end), ask for that in the system prompt.
- **Cache more**: minimum cacheable prompt is 512 tokens, so short system prompts and tool sets now qualify. Changing top-level effort between requests invalidates the cache; use per-message effort for one-off turns.

### Refusals and fallback

- Sonnet 5.5 matches or beats Sonnet 5 on most alignment/honesty audit measures and is the first Sonnet with cybersecurity safeguards like the top models; routine development is mostly unaffected.
- A decline is HTTP 200 with `stop_reason: "refusal"`; `stop_details` names one of `cyber`, `bio`, `frontier_llm`, `reasoning_extraction`, `general_harms`.
- Server-side fallback (`fallbacks: "default"`, beta, Claude API) re-sends `cyber` and `frontier_llm` declines to Sonnet 5; the other three categories get no retry. Alternatively use SDK middleware or your own retry. The Cyber Verification Program is slated to cover Sonnet 5.5 for legitimate security work.

### Sonnet 5.5 in Claude Code

From v2.1.284 (Agent SDK TypeScript v0.3.284+), `sonnet` → Sonnet 5.5 on the Claude API, at `medium` effort by default with native 1M context. Thinking can't be turned off there; effort sets how much it thinks. No fast mode. The default model stays Opus 5.5; use `/model sonnet` for well-scoped tasks.

---

## 11. Compatibility checklist before migrating

All current models support tool use and image input, but a model swap is not always just a name change. Test these:

- **Fable 5.2 preserved thinking** (now on for every API account): the API returns 400 if anything *before* a Fable 5.2 thinking block changed (system prompt, tools, an earlier message) unless you opt in to dropping the affected thinking. Claude Code, the Agent SDK and Managed Agents handle this for you. If your own code edits history, make it append-only and test before switching.
- **Thinking**: Opus 5.5 and Fable 5.2 always think; turning thinking off errors. Sonnet 5.5 thinks by default; `between_tools` limits it to gaps between tool calls. Haiku 5.5 thinks by default and can be turned off at `high` or below. On all of them, read content blocks by type: a response may begin with a thinking block, and code that reads `content[0].text` breaks.
- **Forced tool use**: `tool_choice` `any` or `tool` → 400 on Sonnet 5.5, Opus 5.5 and Fable 5.2. Use `auto` instead, set `strict: true` on the tool, and tell the model in the prompt when to call it.
- **Computer use**: supported on Opus 5.5, Sonnet 5.5, Haiku 5.5 and Fable 5.2, but only via `computer_toolset_20260801` on the Claude API and Claude Models on Google Cloud (Google managed). If your code still declares `computer_20251124`, treat that as a breaking change to fix before moving to Opus 5.5 or Fable 5.2.
- **Fable 5.2 extras**: check before switching if your code moves conversations between models or uses the advisor tool (its executor only accepts Fable 5.2 or Mythos 5.2 advisors).
- **Sonnet 5 → 5.5**: see section 10 for all six steps. In short: `thinking: {"type": "disabled"}` → 400, use `{"type": "between_tools"}` (low/medium/high only, no other fields, effort fixed for the conversation; adaptive otherwise); forced `tool_choice` → `auto` + strict; append-only history; `computer_20251124` and the `fine-grained-tool-streaming` beta header → 400 alongside the toolset, use `computer_toolset_20260801` and per-tool `eager_input_streaming: true`; Opus 4.8, Opus 4.7 and Sonnet 5 are rejected as advisors to a Sonnet 5.5 executor; interim progress text arrives as thinking blocks, pass them back unchanged. `/claude-api migrate this project to claude-sonnet-5-5` applies these in Claude Code.
- **Haiku 4.5 → Haiku 5.5**: replace `budget_tokens` with an effort level; drop non-default `temperature`, `top_p`, `top_k` (400); end with a user turn (a trailing assistant prefill → 400).
- **Opus 5 → Opus 5.5**: default effort drops from `high` to `medium` (omitted effort now runs one level lower); thinking can't be disabled; don't reuse old effort levels.
- **Safety fallbacks**: Opus 5.5, Sonnet 5.5 and Fable 5.2 run classifiers that mostly flag cybersecurity and biology content. In Claude Code a flagged request may re-run on an older model with a transcript notice. Fable 5.2 keeps Fable 5.1's fallbacks: cyber → Opus 4.8, biology → Opus 5.
- **Multi-model systems**: build for the strictest model (append-only history, `tool_choice: auto` + strict tools, read blocks by type, literal-model-proof prompts). Route in code when the rule is simple.
- **Stale IDs**: search the codebase for hard-coded `claude-` model IDs and compare them against the ID table in section 2.

---

## 12. Apps that host an agent (Agent SDK, Claude Code, ACP)

When your app runs an agent rather than calling the API itself:

- **Default to the agent's own default model** (Opus 5.5 in Claude Code) instead of pinning an ID, and let the user override it. In the Agent SDK, `setModel("default")` or `setModel(undefined)` returns to Claude Code's default; `ANTHROPIC_MODEL` in the agent's environment sets the starting model.
- **Accept aliases.** Pass `opus`, `sonnet`, `haiku` (and `fable`) through to the agent rather than rejecting anything that isn't an exact ID; the agent resolves them.
- **Handle a refused model visibly.** The Agent SDK's `setModel()` (Claude Code v2.1.268+) refuses an ID the provider doesn't offer, and ACP adapters may reject a value they didn't advertise. Show the error and fall back to the default model; never drop the choice silently.
- **Expose effort for Claude models.**
  - Agent SDK: the `effort` option of `query()` (`low`, `medium`, `high`, `xhigh`, `max`), or `applyFlagSettings({ effortLevel })` mid-session; `effortLevel: null` returns to the model's default.
  - Claude Code CLI: `--effort`, `/effort`, or `CLAUDE_CODE_EFFORT_LEVEL` (which overrides both).
  - ACP: the adapter's effort config option; offer only the levels the current model advertises.
  - Raw API: `output_config.effort`. Opus 5.5 and Haiku 5.5 default to `medium`, most others to `high`. On the raw API, sending the default level is the same as omitting it.
- **Don't pin effort by accident.** In an agent, an explicit level is a user choice that overrides the user's own settings and can survive model switches. To follow the user's settings, omit it (or send the adapter's "default" value). Never let a global default overwrite a per-session or per-model choice, and never let another provider's default leak into Claude turns.
- **Send model, effort and fast mode only when they change.** Re-sending an unchanged value doesn't break the cache, but changing it can: a model switch always starts a new cache; a top-level effort change does on the raw API and on Bedrock/Google Cloud Agent Platform/gateways; turning fast mode on the first time does.
- **Fast mode** (Opus 5.5, Fable 5.2) is reachable from apps: the `fastMode` setting (`applyFlagSettings` in the SDK), or the adapter's fast-mode option over ACP. It keeps a separate cache, so set it at the start of a session.
- **Subagent models.** `CLAUDE_CODE_SUBAGENT_MODEL` is read by Claude Code itself; when your app launches the agent, put it in the agent process's environment (plus `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` to make it override definitions). Or set `model:` in each agent definition. Read/search subagents on Haiku or Sonnet 5.5.
- **Look for stale IDs.** Search the code for hard-coded `claude-` model IDs and compare them with the ID table in section 2 before migrating.

---

## 13. Measure before committing

Published benchmarks pick a starting point; your own tasks decide.

1. Sample real tasks. Write an outcome check for each (passing test, closed ticket, correct row count).
2. Run each candidate model at 2–3 effort levels, each in its own session so runs don't share a cache. Record each completed task's cost from the response `usage` field.
3. Chart score against spend. A multi-model setup stays only if it beats every single-model point on that chart.
4. Shadow-run the winner on a slice of real traffic before cutting over.

Include the hardest tasks in the sample (they dominate spend).

**Quick check in Claude Code**: pick a real backlog task (not a toy), run it on two models via `/model`, compare `/usage` (alias `/cost`). Note turns, output tokens and cost; do 3–4 tasks before concluding. Also try the effort ladder: one hard task at `medium` then `high`; one mechanical task at `low`. Update Claude Code first (`claude update`; Opus 5.5 needs v2.1.280+). For teams, the Claude Code Analytics API gives estimated cost per user, and the Usage and Cost API splits spend by model and by cached vs uncached tokens.

**Reading a `/usage` session.** The Session block shows input, output and cache figures plus an estimated list-price dollar cost (on subscriptions also plan-usage bars; the dollar figure is a guide, not a bill). Check:

1. **Cache share**: should be high on a long session. If low, look for a long pause, a model switch, an MCP server connected mid-session, or (on a cloud provider/gateway) an effort change. The prompt-cache line often names the likely cause of the last miss.
2. **Output vs input**: lots of output on a small change usually means effort is too high for the task, or the model is retrying.
3. **Total input vs conversation size**: if total input is many times the conversation's size, the session took many turns; read it to find where the loop repeated.

---

## 14. How to answer the user

1. Name the model and effort for their task, with a one-line reason from the tables.
2. Say what would make them move up or down (and remind them to try verification and effort before switching model).
3. If they are integrating via the API, mixing models, or migrating, flag the relevant compatibility items from section 11.
4. If cost matters, give the relevant numbers from sections 6–7 and suggest the measurement loop in section 13 rather than trusting list prices.

---

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
