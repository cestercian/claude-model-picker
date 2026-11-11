---
name: prompt-caching
description: Design or audit an agent so Claude prompt caching keeps hitting - raw Messages API harnesses and apps built on the Claude Agent SDK, Claude Code, or ACP. Use when building an agent loop, when Claude API costs or latency are higher than expected, when the cache hit rate is low, or when adding plan modes, model or effort switching, dynamic or MCP tools, timestamps, process restarts, or conversation compaction.
---

# Prompt caching for agents

Everything you need is in this file. Nothing here requires fetching a page.

## The one constraint

Prompt caching is a **byte-for-byte prefix match** over the request, in the order `tools` → `system` → `messages`. The API stores the processed state of a prefix; a later request reuses it only if its own prefix is identical from the very first token up to a cache breakpoint. Change one byte anywhere and everything after that byte is recomputed and re-billed. There is no per-file or per-segment caching.

Design the whole harness around that constraint. If static content comes first and the conversation only ever grows at the end, most caching works with no further effort. Nearly every cache bug is something that edits the front or middle of the prompt.

Why it matters so much for agents: an agent loop resends the entire conversation on every turn. On Opus 5.5 (API list prices: $4/MTok input, $0.20/MTok cache read, $20/MTok output), a task whose context grows from 20K to 120K tokens over 40 turns sends about 2.8M input tokens in total:

| Cache hit rate | Input cost for that task |
|---|---|
| 0% | $11.20 |
| 90% | $1.62 |
| 96% | $0.99 |

No other setting moves input cost this much. That is why the Claude Code team alerts on hit rate and declares an incident (a SEV) when it drops: a few points of miss rate changes cost and latency dramatically, and a higher hit rate is what lets subscription plans offer more generous limits.

## First: who builds the requests?

- **Your code calls the Messages API.** Everything in this file applies.
- **Your app drives the Agent SDK, Claude Code, or an ACP agent.** The SDK already orders the prompt, places `cache_control`, injects updates as messages, and compacts with a cache-safe fork. Skip those mechanics. What your app still controls, and can still break, is in [Apps built on the Agent SDK, Claude Code, or ACP](#apps-built-on-the-agent-sdk-claude-code-or-acp). Read that section and [Auditing](#auditing) before auditing such an app.

---

## How the API cache works

### Enabling it

Caching is opt-in on the Messages API. A request with no `cache_control` anywhere caches nothing and pays full input price for the whole conversation every turn. (Managed Agents caches automatically with a 5-minute lifetime.)

Two ways to turn it on:

- **Automatic caching.** One top-level field: `"cache_control": {"type": "ephemeral"}`. The API puts the breakpoint on the last cacheable block and moves it forward as the conversation grows, so you never update markers. Best default for multi-turn agents.
- **Explicit breakpoints.** Put `cache_control` on individual blocks (a tool, a `system` block, a message content block). Use when sections change at different rates or the last block varies per request.

You can combine them: the automatic breakpoint takes one of the 4 breakpoint slots. Automatic caching edge cases:
- Last block already has an explicit `cache_control` with the same TTL: no-op.
- Last block has an explicit `cache_control` with a different TTL: 400 error.
- 4 explicit breakpoints already present: 400 error (no slot left).
- Last block not eligible as a target: the API walks backward to the nearest eligible block, or skips caching if none.
- Not available on the legacy Amazon Bedrock integration (Opus 4.6 and earlier), which returns 400 for a top-level `cache_control`; use explicit breakpoints there.

`ephemeral` is the only cache type.

### What a breakpoint actually does

1. **Writes happen only at breakpoints.** A breakpoint writes one entry: a hash of the whole prefix ending at that block. No entries are written for earlier positions. Because the hash is cumulative, changing any block at or before the breakpoint changes the hash.
2. **Reads look backward for prior writes.** The API hashes the prefix at your breakpoint; if there's no entry, it walks back one block at a time looking for an entry an *earlier request wrote*. It does not discover stable content on its own.
3. **The lookback window is 20 positions** per breakpoint, counting the breakpoint itself. On the Claude API, back-to-back `tool_use` blocks are counted as a single position, and so are back-to-back `tool_result` blocks, which means a turn full of parallel tool calls doesn't push the last entry out of range by itself.

Consequences:

- **Growing conversation:** a breakpoint on the last block keeps working while every turn adds under 20 blocks. Turn 1 writes at block 10; turn 2 (15 blocks) walks back from 15, hits 10, processes 11-15 fresh, writes at 15. If turn 3 jumps to 35 blocks, the lookback checks 35 down to 16 and misses the entry at 15. Fix: add a second breakpoint from the start near where the jump lands, so an entry exists there before you need it.
- **Classic mistake: breakpoint on a varying block.** Static context in blocks 1-5, then block 6 holds a timestamp plus the user message, breakpoint on block 6. Every request writes a fresh entry at 6 (hash includes the timestamp), the lookback walks 5..1 and finds nothing because nothing was ever written there. You pay a cache write every request and never read. Move the breakpoint to block 5, the last block identical across requests. Automatic caching has the same failure here because it targets the last block, so use an explicit breakpoint on block 5.
- **Rule:** place `cache_control` on the last block whose prefix stays identical across every request that should share the cache.

Up to **4 breakpoints** per request. Breakpoints are free; you pay only for tokens written and read. Use several when sections change at different rates. The classic four-breakpoint layout:
1. Last tool definition (caches all tools).
2. End of static system instructions.
3. End of RAG / document context (can change daily without invalidating 1-2).
4. Last message (incremental conversation caching).

A change at any breakpoint invalidates that segment and everything after; earlier segments stay valid.

### Minimum cacheable length

Shorter prefixes are silently not cached (no error). If both `cache_creation_input_tokens` and `cache_read_input_tokens` are 0, you are probably under the minimum.

| Minimum | Models |
|---|---|
| 512 tokens | Fable 5.2, Mythos 5.2, Fable 5.1, Mythos 5.1, Opus 5.5, Opus 5, Sonnet 5.5, Fable 5, Mythos 5, Haiku 5.5 |
| 1,024 tokens | Opus 4.8, Sonnet 5, Sonnet 4.6, Sonnet 4.5, Opus 4.1, Opus 4, Sonnet 4 |
| 2,048 tokens | Mythos Preview, Opus 4.7 |
| 4,096 tokens | Opus 4.6, Opus 4.5, Haiku 4.5 |

These apply on the Claude API, Claude Platform on AWS, Google Cloud (Google managed) and Microsoft Foundry. Bedrock publishes its own per-model minimums, failure behavior and field names. If a frequently reused prompt is just under the minimum, padding it up to the threshold is often worth it, since reads are so much cheaper than fresh input.

### What can and can't be cached

Cacheable: tool definitions; `system` blocks; text blocks in user and assistant turns; images and documents in user turns; `tool_use` and `tool_result` blocks.

Not directly cacheable:
- **Thinking blocks** can't carry `cache_control`, but they are cached as part of the prefix when they sit in earlier assistant turns (typically when you pass them back with tool results). Read from cache, they count as input tokens.
- **Sub-content blocks** such as citations: cache the top-level document block instead.
- **Empty text blocks.**
- **Deferred tools** (`defer_loading: true`) can't carry `cache_control` (400). Put the breakpoint on a non-deferred tool.

### Concurrency, scope, and isolation

- An entry becomes available only **after the first response starts streaming**. For parallel requests that should share a prefix, send one, wait for its response to begin, then send the rest. (Claude Code holds same-prefix workflow fan-out agents up to 5 seconds for exactly this reason.)
- Caches are never shared across organizations. On the Claude API, Claude Platform on AWS and Microsoft Foundry they are also isolated per **workspace**; Bedrock and Google Cloud (Google managed) isolate per organization only. Two requests share a cache only when model, speed (fast vs standard) and prefix all match.
- Caching never changes the output; responses are identical with or without it.
- There is no manual cache clear; entries expire.
- Message Batches: caching works but hits are best-effort because batch items run concurrently in any order. Best pattern: send one request with the shared prefix and a 1-hour breakpoint, wait for it to finish, then submit the rest.

### Pre-warming

To remove the cold-start penalty on a user's first request, send a request with `max_tokens: 0`. The API processes the prompt, writes the cache at your breakpoints, and returns at once with empty `content`, `stop_reason: "max_tokens"`, full `usage`, and zero billed output.
- Put an **explicit** breakpoint on the last block shared with the real follow-up (system prompt or tools), not on the placeholder user message (any non-whitespace string). Automatic caching would key the entry to the placeholder.
- Use the same thinking config and `output_config.effort` as real traffic; both are rendered into the prompt, so a mismatched warm-up writes an entry nobody hits.
- Rejected with `max_tokens: 0`: `stream: true`, `thinking.type: "enabled"`, `output_config.format`, `tool_choice` of `tool` or `any`, and any Message Batches request.
- Re-warm at least every 5 minutes on the 5-minute TTL, or use the 1-hour TTL.
- Prefer this over the older `max_tokens: 1` trick.

---

## Pricing and TTL

### Multipliers

| Token kind | Price |
|---|---|
| 5-minute cache write | 1.25 × base input |
| 1-hour cache write | 2 × base input |
| Cache read (hit or refresh) | 0.1 × base input on most models; **0.05 ×** on Opus 5.5 and Sonnet 5.5; **0.025 ×** on Fable 5.2, Mythos 5.2, Fable 5.1, Mythos 5.1 |
| Uncached input after the last breakpoint | 1 × base input |

These stack with the Batch discount, long-context pricing and data residency. On models priced by prompt length (Haiku 5.5), cache reads and writes count toward the prompt length.

Per-million-token prices for current models:

| Model | Base input | 5m write | 1h write | Cache read | Output |
|---|---|---|---|---|---|
| Fable 5.2 / Mythos 5.2 | $8 | $10 | $16 | $0.20 | $40 |
| Fable 5.1 / Mythos 5.1 | $10 | $12.50 | $20 | $0.25 | $50 |
| Fable 5 / Mythos 5 | $10 | $12.50 | $20 | $1 | $50 |
| Opus 5.5 | $4 | $5 | $8 | $0.20 | $20 |
| Opus 5 / 4.8 / 4.7 / 4.6 / 4.5 | $5 | $6.25 | $10 | $0.50 | $25 |
| Sonnet 5.5 | $2 | $2.50 | $4 | $0.10 | $10 |
| Sonnet 5 | $2 | $2.50 | $4 | $0.20 | $10 |
| Sonnet 4.6 / 4.5 | $3 | $3.75 | $6 | $0.30 | $15 |
| Haiku 5.5 (prompt ≤ 100K) | $0.10 | $0.125 | $0.20 | $0.01 | $0.50 |
| Haiku 5.5 (prompt > 100K) | $0.50 | $0.625 | $1 | $0.05 | $2.50 |
| Haiku 4.5 | $1 | $1.25 | $2 | $0.10 | $5 |

Fast mode (Opus 5.5) is twice standard: $8 input, $40 output.

What this means in practice on Opus 5.5 at 120K tokens of context: a 5-minute write runs about $0.60 and a read about $0.02. **A single write costs the same as 25 reads.** With an API key on the 5-minute TTL, stepping away for six minutes for coffee makes the next request a $0.60 write instead of a $0.02 read. A 1-hour write of the same context is about $0.96.

Output dominates per-token cost: an Opus 5.5 output token costs 100 × a cache read, and thinking is billed as output. So caching fixes input cost; effort and turn count fix output cost.

### Cache lifetime

- Default TTL is **5 minutes**. Every hit refreshes the lifetime at no charge, so a busy conversation stays warm indefinitely.
- The lifetime is measured **from the start of the request** that wrote or read the entry, not from the end of its response. Generation time counts. If a response streams for 4 minutes, the next request has about 1 minute to start.
- 1-hour TTL: `"cache_control": {"type": "ephemeral", "ttl": "1h"}`. Available on the Claude API, Bedrock (both integrations), Claude Platform on AWS, Google Cloud (Google managed) and Microsoft Foundry. Latency is the same as the 5-minute cache.

**Choosing a TTL.** If the prefix is reused more often than every 5 minutes, stay on 5 minutes: it refreshes free and writes are cheaper. Use 1 hour when:
- gaps between turns are typically 5-60 minutes (a user who may not reply right away, a side agent that runs longer than 5 minutes, long responses that eat the window);
- latency matters and follow-ups may arrive after 5 minutes;
- you want better rate-limit utilization (cache hits don't count against rate limits).

**Mixing TTLs** in one request is allowed only if 1-hour entries come **before** 5-minute entries. Billing then splits into: A = tokens up to the last cache hit (billed as reads); B = last 1-hour breakpoint after A (B−A billed as 1-hour writes); C = last breakpoint (C−B billed as 5-minute writes). A hit on a 5-minute entry doesn't count if a later 1-hour breakpoint misses, and reading a 5-minute entry restarts its 5-minute life even if the request now asks for `1h`.

---

## Lay out the prompt for caching

Order from most stable to least stable, so the maximum number of requests (across sessions, not just within one) share a prefix. Claude Code's layout:

1. **Static system prompt and tool definitions** - identical for every user everywhere, so cached globally.
2. **Project context** (CLAUDE.md, memory, rules) - identical within a project.
3. **Session context** (working directory, platform, git snapshot) - identical within a session.
4. **Conversation messages** - grow at the end each turn.

This ordering is fragile. Claude Code itself broke it by:
- putting a detailed timestamp in the static system prompt (a different prefix every second or minute);
- serializing tool definitions in a non-deterministic order (shuffled tool list = different prefix);
- changing a tool's parameters based on state, e.g. updating which agents the Agent tool can call.

Each of these makes the very first block differ, which costs the whole conversation.

---

## Rules (each with its reason)

### 1. Put updates in messages, not in the system prompt or tools

**Why:** the system prompt sits near the front of the hashed prefix. Editing it, even appending one sentence, invalidates the system cache and every cached message after it, so the next turn re-bills the whole history. Information that goes stale (current time, a file the user changed, a mode switch, remaining budget) should be appended at the end instead, where it only adds a few new tokens.

How, in order of preference:

**a) Mid-conversation system message** (no beta header). Append `{"role": "system", "content": "..."}` to `messages` at the point the instruction starts to apply.
- Models: Fable 5.2, Mythos 5.2, Fable 5.1, Mythos 5.1, Fable 5, Mythos 5, Mythos Preview, Opus 5.5, Opus 5, Opus 4.8, Sonnet 5.5, Haiku 5.5. **Not Sonnet 5** (use the top-level `system` there and accept the cost, or use option b).
- Platforms: Claude API, Claude Platform on AWS, Google Cloud (both), Bedrock. Managed Agents: send a `system.message` event (the agent's own `system` can't change mid-session).
- It keeps **operator** priority, unlike text in a user turn: later system messages override earlier ones, and override the top-level `system` for subsequent turns.
- Placement: a system message with content must directly follow a `user` turn (a `tool_result`-only user turn counts) or an assistant turn ending in a server tool result, and must be last or directly followed by an `assistant` turn. It can't be first in `messages`, and can't sit between a `tool_use` and its `tool_result` (400). Consecutive system messages are treated as one section.
- Once sent, it is part of the stable history and is itself cacheable; move the breakpoint past it next turn (automatic caching does this).
- **Never edit or delete one after sending** - that's an edit to earlier messages, invalidating from there on, and on Fable 5.2, Mythos 5.2, Fable 5.1, Mythos 5.1, Opus 5.5, Sonnet 5.5 and Haiku 5.5 it also invalidates thinking blocks in every later assistant turn. To evolve an instruction, append a new one.
- Phrase it as a fact ("the user sent X while you were working", "remaining budget is Y"), not as "ignore the user".
- **Never put untrusted content** (tool output, web pages, retrieved documents) in a system message: it gets operator authority. Keep that in `tool_result` blocks.
- On **Sonnet 5.5 and Haiku 5.5**, pass along user input typed mid-loop as a `text` block placed after the final `tool_result` inside the same `user` message, not as a system message after tool results (these models may treat a system message right after tool results as an injection and ignore it). Keep harness notices in a separate system message.

**b) A tagged block in the next user message or tool result.** Works on every model. Claude Code appends a `<system-reminder>...</system-reminder>` block to the next user message or tool result (for example, "file X changed since you read it"). Lower priority than a system message, but the cache is untouched.

**c) Turn-scoped reminders that don't pile up** (beta header `mid-conversation-system-clear-at-2026-08-21`). Set `"clear_at": "next_user_message"` on a system message. It renders only until a later `user` message (including a tool-result-only one) exists; after that it stays in the array but renders nothing and costs nothing. Use it for per-loop nudges ("batch independent reads"). Rules: text only (no `tool_addition`, `tool_removal`, `output_config`); **no `cache_control` on it** (put the breakpoint on the preceding user turn; automatic caching skips these); resend cleared messages **verbatim** forever (rebuilding, dropping, or changing `clear_at` is an edit to history and breaks later thinking blocks); a turn-scoped message directly followed by another user message is a 400, so put all a round's tool results in one user message and reminders after it.

What Claude Code delivers as messages for this reason: plan mode instructions, skill and command instructions, output-style switches, file-change notices.

### 2. Don't switch models mid-session

**Why:** caches are per model. Switching sends the entire history to a model with an empty cache, which pays a full cache write (1.25 × its input price) on all of it, plus the latency of reprocessing, even though the content is identical. Meanwhile the current model would have read the same history at its cheap read rate.

The counterintuitive consequence from the Claude Code team: 100K tokens into an Opus session, asking an easy question is cheaper on Opus than on Haiku, because Haiku must first rebuild a 100K cache. Run the numbers with current prices before assuming the small model saves money: compare `history × new model's write price` against `history × current model's read price`, plus the output difference. (Against Haiku 4.5 that is $0.125 to write 100K vs $0.02 for Opus 5.5 to read it, about 6×. Very cheap targets such as Haiku 5.5 can flip the token math, but you still pay rebuild latency, and if you switch back after the TTL the main model's cache is cold too.)

**Do instead: hand off through a subagent.** Have the main model write a short hand-off message describing the task, and start a subagent on the cheaper model with just that. The subagent's prompt is small, so its cold start is cheap, and the parent's cache is untouched: from the parent's side the subagent call and its result are just appended messages. Claude Code's Explore agents run on Haiku this way.

Disguised model switches to watch for:
- an "auto" or plan/execute model setting (Claude Code's `opusplan` runs Opus in plan mode and Sonnet otherwise, so every plan-mode toggle is a switch);
- a skill or command whose frontmatter names a different `model` (a switch for that turn, and again when the session model resumes);
- automatic fallback to another model after a safety classifier flags a request;
- turning on fast mode on a model that doesn't support it, which switches models.

If you must switch, do it at a natural break, and shrink the history first (compact, or start fresh with a short written plan) so the write on the new model is small.

### 3. Hold effort and thinking settings constant, or change them the cache-safe way

**Why:** the thinking configuration (mode, and `budget_tokens` in extended mode) and the top-level `output_config.effort` are rendered into the prompt. Changing either always invalidates the message cache, and on some models the tools and system caches too.

- Setting effort explicitly to the model's default is identical to omitting it and doesn't invalidate. Defaults: `medium` on Opus 5.5 and Haiku 5.5, `high` on every other model.
- **Per-message effort (beta, header `mid-conversation-output-config-2026-07-01`)** keeps the cache: append a `role: "system"` message with empty `content` and `"output_config": {"effort": "high"}`. Supported on Fable 5.2, Mythos 5.2, Fable 5.1, Mythos 5.1, Opus 5.5, Opus 5, Sonnet 5.5 and Haiku 5.5 (platform coverage varies: Opus 5 not on Bedrock). An effort-only message can go anywhere in `messages`. Placed right after a user turn with new input it applies to that reply; elsewhere it applies from the next user turn with new input (a tool-result-only turn doesn't count). Not usable on Sonnet 5.5 with `between_tools` thinking or Haiku 5.5 with thinking disabled (400 if the level differs).
- On models without per-message effort, pick one level at session start and keep it.
- In Claude Code (and so the Agent SDK), changing effort keeps the cache on Fable 5.1+, Opus 5.5, Sonnet 5.5 and Haiku 5.5 with an API key or Claude subscription. It still invalidates on other models, on Bedrock, Google Cloud's Agent Platform, a Claude apps gateway, with `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS`, or under a HIPAA configuration. There, change effort only at natural breaks.
- On models before Opus 4.5 / Sonnet 4.6 / Haiku 5.5, adding non-tool-result user content strips earlier thinking blocks and invalidates the messages after them. On Opus 4.5+, Sonnet 4.6+ and Haiku 5.5 thinking is preserved and the cache holds.

### 4. Never add or remove tools mid-session

**Why:** tool definitions are the very first thing in the prefix. Adding, removing, reordering or editing any tool (name, description, parameters) invalidates the tools, system and messages caches: the whole conversation. It feels natural to give the model only the tools it needs right now, and it is one of the most common ways people break caching.

Model state changes so the tool list never moves:

**Plan mode as tools, not as a tool swap.** The obvious design (on entering plan mode, swap in a read-only tool set) breaks the cache on every toggle. Claude Code instead keeps every tool in every request and adds two tools, `EnterPlanMode` and `ExitPlanMode`. When the user turns plan mode on, the agent receives a system message saying it's in plan mode and what that means: explore the codebase, don't edit files, call `ExitPlanMode` when the plan is ready. Disallowed calls are refused when they run, not hidden. Bonus: because `EnterPlanMode` is a tool, the model can enter plan mode on its own when it judges a problem hard, with no cache break. Apply the same pattern to any mode (read-only, review, safe mode): a pair of state tools plus a message with the rules, enforced at execution time.

**Defer instead of remove (tool search).** Claude Code may have dozens of MCP tools; sending all full schemas every request is expensive, and removing them breaks the cache. Instead, tools are sent with `defer_loading: true`; the model discovers them through a search tool and only then is the full schema loaded. The prefix stays stable because the same deferred set is always present in the same order. On the API:
- Add a search tool to `tools`: `{"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"}` (Python regex, max 200 chars) or `{"type": "tool_search_tool_bm25_20251119", "name": "tool_search_tool_bm25"}` (natural language, max 500 chars).
- Send **every** tool's full definition in `tools` on **every** request, deferred ones included; `defer_loading` controls what enters the context, not what you send. The API excludes deferred tools from the system-prompt prefix and appends a `tool_reference` inline in the conversation when one is discovered, which is why discovery doesn't disturb the cache.
- At least one tool must be non-deferred (normally the search tool; never defer it). Keep your 3-5 most-used tools non-deferred so they need no search. Put the cache breakpoint on a non-deferred tool.
- Pass back the `server_tool_use` and `tool_search_tool_result` blocks unchanged; never send a `tool_result` for a `srvtoolu_...` id. Discovered tools stay available in later turns without searching again.
- MCP connector: set `defer_loading` once on the `mcp_toolset`'s `default_config`, or per tool in `configs`.
- Custom search: your own tool returns a `tool_result` whose content is `{"type": "tool_reference", "tool_name": "..."}` blocks; each must exist in `tools`, normally deferred.
- Limits: 10,000 deferred tools; 5 results per search by default (`limit` 1-10,000). Supported on all current models from Opus/Sonnet 4.5 and Haiku 4.5 up; not Opus 4.1 or earlier. Not metered separately; loaded definitions count as input. On Bedrock only via InvokeModel.
- Worth it with 10+ tools, more than 10K tokens of definitions, several MCP servers, or a growing library. A five-server setup can spend about 55K tokens on definitions; search typically cuts that by over 85%, and selection accuracy degrades past 30-50 loaded tools anyway.

**Add tools inline (beta, header `inline-tools-2026-09-15`).** For a tool unknown at the first request or whose schema changes later, append a system message with a `tool_addition` block carrying the definition and leave `tools` exactly as first sent:
```json
{"role": "system", "content": [{"type": "tool_addition", "tool": {"type": "tool_definition",
  "definition": {"name": "db_query", "description": "...", "input_schema": {"type": "object", "properties": {"sql": {"type": "string"}}, "required": ["sql"]}}}}]}
```
- Only the appended message is new input; the prefix still matches. Resending an identical definition is a no-op (safe on retry). A new definition under the same name replaces the old one from that point.
- `tool_addition`/`tool_removal` with `{"type": "tool_reference", "name": "..."}` offers or withdraws a tool already declared in `tools` (a deferred tool stays withheld until added). The by-reference form also works with header `mid-conversation-tool-changes-2026-07-01` on more platforms.
- Declare tools you know at the start in `tools` (deferred if not yet offered); define inline only what's unknown or changing.
- Keep at least one non-deferred tool in `tools`. If none, the first inline definition changes the start of the rendered prompt and costs one full cache miss. A tool search tool counts as non-deferred.
- `cache_control` goes on the block or inside the definition, not both; a deferred definition can't carry it.
- Add an MCP server mid-conversation with `mcp-client-2026-09-15` plus `inline-tools-2026-09-15`: the `definition` is `{"type": "mcp_toolset", "mcp_server_name": "..."}`; URLs and tokens stay in `mcp_servers`. Send back the `mcp_tool_listing` block the response starts with, unchanged, so later requests reuse the recorded list.
- Not accepted directly after a paused assistant turn; computer and browser use tools can't be defined inline.
- Models: the same as mid-conversation system messages.

**Stable serialization.** Emit tools and JSON schemas in a fixed order with stable key order. Languages and libraries that serialize maps in random order (Swift dictionaries, Go maps) break the cache without any visible change; this also applies to keys inside `tool_use` blocks you send back.

**State-dependent tools.** If a tool's description or enum lists something that changes (available agents, open files, connected devices), the tool list changes. Move that state into a message or a tool result, and keep the definition fixed.

### 5. Forks and side computations must share the parent's prefix

**Why:** any side call (compaction, summarization, a skill run, a review pass) that uses its own system prompt or tool list shares nothing with the parent and pays full price for whatever history it sends. Build such calls as a copy of the parent's last request with the extra instruction appended at the end. Claude Code's forked subagents inherit the parent's system prompt, tools and history exactly, so their first request reads the parent's cache; a regular subagent with its own prompt does not (that's fine when it starts small).

### 6. Pick the TTL for your idle gaps

**Why:** once an entry expires, the next turn pays a full write on the whole context. If users or agents routinely pause longer than 5 minutes, every resume pays that write; the 1-hour TTL costs 2× instead of 1.25× per write but survives the gaps. See [Pricing and TTL](#pricing-and-ttl) for the break-even and [Claude Code TTL controls](#cache-lifetime-in-claude-code-and-the-agent-sdk).

---

## Compaction without breaking the cache

### What compaction is

When the context window fills, the agent summarizes the conversation so far and continues with the summary in place of the old messages. It also keeps later turns cheaper and quality higher, because every turn resends everything, including leftovers like an hour-old stack trace.

By design compaction invalidates the conversation layer (the new history is shorter and shares no message prefix with the old). The question is what the summarization request itself costs.

### The trap: a separate "summarize this" call

The simple implementation is a separate API call with its own short system prompt ("summarize this conversation") and no tools, sending the whole conversation. Its prefix diverges from the main conversation at the very first token (different tools, different system), so **none** of the cache applies. You pay full uncached input for the entire conversation, and the longer the conversation (the more you needed compaction), the more that one call costs.

### Cache-safe forking, step by step

1. Take the parent's exact last request: same `tools` (same order and bytes), same `system`, same user/system context blocks, same model, same thinking and effort settings.
2. Keep all of the parent's messages as they are.
3. Append the compaction instruction as one new user message at the end (what to keep, "do not call tools", return only the summary).
4. Send it while the cache is still warm. To the API this looks like the parent's next turn, so it reads the whole prefix from cache; only the instruction and the summary output are new tokens.
5. Replace the history with the summary and continue. The next turn writes a cache for the much shorter history.
6. **Reserve a compaction buffer:** trigger compaction early enough that the context window still has room for the instruction plus the summary's output tokens. If you wait until the window is full, there is no room to ask.

Claude Code works this way: its summary request repeats the conversation's system prompt, tools and history and ends with the instruction, so a mid-session `/compact` reads the prefix from cache and spends most of its time generating. After compaction the system prompt layer is read from cache, and the project context layer is too if CLAUDE.md and memory haven't changed since the session started.

### Costs and timing (Opus 5.5 illustration)

- Compacting at 150K tokens with a warm cache costs about **$0.25** (the cached read, a few thousand output tokens of summary, a new cache write on the short history). Each later turn saves about $0.025 in reads, so it pays for itself in about **ten turns**. Compacting just before finishing costs more than it saves.
- With a cold cache (after a break longer than the TTL) the same compaction reprocesses the history uncached: about **$0.75** of input alone at 150K. **Compact before a break, not after one.** This is also why compacting a resumed old session costs the most.
- Compact at a natural break between tasks rather than letting auto-compaction fire mid-task, and say what must survive (a summary can drop the one log line that mattered).
- To abandon a wrong path, **rewind** (truncate back to an earlier turn) instead of compacting: the remaining history is an already-cached prefix, and every turn since kept it warm.
- To start unrelated work, clear the conversation entirely; that costs nothing.
- A display-only recap that appends output (rather than replacing history) keeps the cache.
- Images, documents, uploaded files and fetched URLs inside summarized messages are gone afterward; restate anything still needed.

### Compaction built into the API

Anthropic built these lessons into the API, so you need no summarizer of your own. Three options:

| | On demand | At a token threshold | Your own summarizer |
|---|---|---|---|
| Who decides when | You, by sending a request | The API, when input reaches your trigger | You |
| Code you write | A loop that requests the summary and swaps it in | One parameter on ordinary requests | The call, its prompt, the history rewrite |
| Recent turns kept verbatim | Yes (keep-recent-turns option) | Yes, by pausing after compaction and reinserting | Yes |
| Runs in background | Yes | No | Yes |
| Kept turns keep their thinking | Yes, under conditions | No (strip it, or `prefix_mismatch_behavior: "drop_block"`) | No (fails the prefix check) |
| Choose when | You need control over timing, can't pause for a summary, or have to keep recent turns along with their thinking | You'd rather the API manage context within ordinary requests | You already have one that replaces the whole history |

Prefer on-demand compaction wherever it's available.

**On-demand compaction** (beta header `compact-2026-09-04`; Fable 5.2/5.1/5, Mythos 5.2/5.1/5/Preview, Opus 5.5, Opus 5, Sonnet 5.5, Sonnet 5, Haiku 5.5, Opus 4.8/4.7/4.6, Sonnet 4.6 on the Claude API; similar on AWS, Bedrock, Google Cloud (Google managed), Foundry; check a model's `capabilities.compaction` via the Models API):

1. After a completed turn, estimate the next request's size: `input_tokens + cache_read_input_tokens + cache_creation_input_tokens + output_tokens` from the last response's `usage` (or call the token counting endpoint). Compare with a limit you set below the context window, leaving the buffer.
2. Send the conversation as it stands with `"compaction": {"type": "summarize"}` and the **same model, `system`, `tools` and thinking settings** as the conversation (the summarizer reads them; matching them is also what keeps the prefix shared and kept thinking valid). Give `max_tokens` several thousand; it caps thinking plus summary.
3. Don't send: `stop_sequences`, `output_config.format`, `tool_choice` of `any`/`tool`, `context_management`, or `output_config.task_budget.remaining`. The last assistant turn must not end in an unanswered tool call.
4. Check `stop_reason` first. `"compaction"` means `content` holds one `compaction` block (`content` text + `signature`). Otherwise the response is a 200 with empty content: `max_tokens` (resend larger), `model_context_window_exceeded` (shorter instructions or fewer messages), `tool_use` (tell it not to call tools), `refusal` or `end_turn` (continue without, try later).
5. Replace the summarized messages with the returned assistant message. The block must be **first** in `messages` (its own assistant message, or the first block of the first message), exactly one block per request, sent byte-for-byte as returned on every later request, with the beta header on every request that carries it. Turns taken while the summary was being written follow the block unchanged.
6. To compact again, send `compaction` again; the new block summarizes the old one plus everything after. Keep only the newest.

Options: `instructions` (non-blank, up to 16,384 characters) replaces the default summarization prompt; say what to keep and to not call tools. Keep-recent-turns compaction summarizes only older turns. Background compaction lets work continue while the summary is written.

Silent mistakes: summarized messages left after the block get sent again; a request missing the block gives Claude no summary. Errors with codes: `compaction_block_misplaced` (summarized messages in front of the block), `compaction_signature_invalid` / `compaction_content_mismatch` (block edited), `compaction_nothing_to_summarize`, 529 `compaction_unavailable` (retry). In Python, use `client.beta.messages`, or serialize with `to_dict()` / `model_dump(exclude_none=True)`; plain `model_dump()` adds `citations: null` and is rejected.

Interactions: `cache_control` on the block puts a breakpoint after the summary. Mid-conversation system messages and per-message effort inside the summarized range stop applying; restate any that still matter in a new system message right after your next new user turn. Billing: top-level `input_tokens`/`output_tokens` are 0; the call is reported in `usage.iterations` (entry type `compaction`), so sum iterations to count usage. Sending the block later costs nothing extra.

The SDK tool runners can do this for you: call `compact_before_next_turn()` (`compactBeforeNextTurn()` / `CompactBeforeNextTurn()`) and create the runner with the `compact-2026-09-04` beta.

---

## What invalidates the cache (API reference)

The hierarchy is tools → system → messages; a change at one level invalidates that level and all after it.

| Change | Tools cache | System cache | Messages cache |
|---|---|---|---|
| Tool definitions (add, remove, reorder, edit name/description/params) | ✘ | ✘ | ✘ |
| Web search toggled | ✓ | ✘ | ✘ |
| Citations toggled | ✓ | ✘ | ✘ |
| Speed (`speed: "fast"` vs standard) | ✓ | ✘ | ✘ |
| `tool_choice` | ✓ | ✓ | ✘ |
| Images added or removed anywhere | ✓ | ✓ | ✘ |
| Thinking parameters (mode, `budget_tokens`) | model-specific | model-specific | ✘ |
| Top-level `output_config.effort` (except setting it to the default) | model-specific | model-specific | ✘ |
| Non-tool-result user content with thinking | ✓ | ✓ | ✓ on Opus 4.5+, Sonnet 4.6+, Haiku 5.5; ✘ on earlier |
| A thinking block the API drops (not preserved) | ✓ | ✓ | ✘ from that block on |
| Mid-conversation system message, per-message effort, inline `tool_addition` | ✓ | ✓ | ✓ |

Also invalidating: any edit to earlier messages (rewriting, trimming, reordering, re-rendering with fresh values), changing the model, and expiry. On models with preserved thinking, altering anything before a thinking block also makes the request fail the thinking prefix check.

---

## Claude Code and the Agent SDK: what keeps and breaks the cache

This is how Claude Code (and so the Agent SDK and ACP adapters built on it) behaves. Use it to predict costs in apps built on them, and as a reference design for your own harness.

### Layers

| Layer | Contains | Changes when |
|---|---|---|
| System prompt | Core instructions, tool definitions | The set of loaded tool definitions changes |
| Project context | CLAUDE.md, auto memory, unscoped rules | Session start, or after clear / compact |
| Conversation | Messages, responses, tool results | Every turn |

Model, speed and (on most models) effort are not layers but still select which cache is used.

### Actions that invalidate

- **Switching models** (`/model`, `setModel()`): next request reads the whole history uncached. Claude Code asks for confirmation only while the cache is warm (within one TTL of the last request or response) and the new model didn't produce the last response. Includes `opusplan` toggles, skill/command frontmatter `model`, and automatic safety fallback (on Fable models, Opus 5.5, Sonnet 5.5, Opus 5). A `context: fork` skill's `model` sets the forked subagent's model instead.
- **Changing effort**: see Rule 3 (kept on current models with an API key or subscription; invalidated elsewhere, with confirmation while warm).
- **Fast mode**: fast and standard keep separate caches. The first time it's turned on, the next request reads the whole conversation uncached at fast-mode prices. Turned on mid-turn, that turn finishes at standard speed and the uncached read lands on the next turn. Every later toggle, including automatic fallback to standard after a rate limit and the return to fast, reads uncached whatever the other speed lacks. Turn it on at session start.
- **Connecting or removing an MCP server**: depends on tool search, which is on by default on supported models.
  - Tools deferred: Claude Code keeps the first request's tool list for the whole conversation; servers connecting later supply deferred definitions loaded on demand. Cache kept.
  - Tools loaded upfront (tool search off or unavailable, e.g. Google Cloud's Agent Platform models older than Opus/Sonnet/Haiku 4.5, or a Foundry deployment hosted on Azure that rejects tool search):

    | Mid-session change | Cache | Next request's tool definitions |
    |---|---|---|
    | Server connects, or a dynamic tool update adds tools | Invalidated | New definitions added |
    | Server drops out on its own (stdio process exits) | Kept | Unchanged; calls error unless the server is restarted for that call |
    | Remote server reconnects automatically | Kept, except a request sent during reconnection may add `WaitForMcpServers` once (invalidates once, then it stays listed) | Unchanged |
    | You remove a tool on purpose (deny rule, disabling the server) | Invalidated | Definition removed |

  - On resume, a still-connecting server's recorded definitions are sent as recorded, so the request doesn't change when it finishes connecting with the same tools.
  - Editing MCP config does nothing until restart.
  - The advisor tool is an exception: its definition sits after the cache breakpoint, so toggling it keeps the cache.
- **Plugins**: skills, commands, agents, hooks, monitors and themes are appended after the conversation and never invalidate. Plugins providing MCP servers follow the MCP rules. A code-intelligence plugin adds the LSP tool. Changes apply on `/reload-plugins` (run when the plugin menu closes); if a reload would force a full re-read, Claude Code warns and holds it (`--force` to apply). In non-interactive sessions (desktop app, Agent SDK, `-p`), a typed reload applies everything except plugin MCP server changes, which wait for the next session. Disabling a plugin enabled earlier in the session restores the old request shape, which reads the older entry if still within TTL.
- **Denying an entire tool** (a bare name like `Bash`, `Bash(*)`, a tool-name glob like `"*"` or `"mcp__*"`): with tool search active the definitions don't change and the cache survives; without it the definition is removed (invalidates) and removing the rule later invalidates again. Scoped deny rules like `Bash(rm *)`, and all allow/ask rules, are checked at call time and never touch the prefix.
- **Compaction**: invalidates the conversation layer by design (see Compaction). Idle compaction can run on long conversations idle under the one-hour TTL; `idleCompaction: false` turns it off.
- **Accumulating images**: when a request would exceed the per-request image/PDF count or total-size limit, Claude Code drops a batch of the oldest. That edits the messages holding them, so the next request reprocesses from the earliest one. Batching means one slow turn per batch, not per screenshot.
- **Upgrading Claude Code**: a new version usually changes the system prompt or tools, so the first conversation after an upgrade builds from the top. Auto-update applies only at next launch, never mid-session. `DISABLE_AUTOUPDATER=1` controls timing.

### Actions that keep the cache

- Editing repository files: reads are appended; edits don't rewrite earlier reads. A `<system-reminder>` notes the change and Claude re-reads if needed.
- Editing root or user CLAUDE.md mid-session: no invalidation, but no effect either until clear, compact or restart (it's read once at start). Nested CLAUDE.md files and path-scoped rules load on demand; edits before they load take effect, edits after don't.
- Changing permission mode (except plan mode under `opusplan`).
- Changing output style: delivered as a message from the next turn.
- Invoking skills and commands: instructions injected as user messages at invocation.
- A display-only recap, and rewinding the conversation.
- Spawning a subagent: from the parent's side it's an appended call and result.

### Resuming a session

The whole conversation is resent; whatever prefix is unchanged and still within TTL is read from cache. By default a resumed conversation keeps the system prompt it started with (even after an upgrade or with different `--append-system-prompt` text), so its history stays behind the same prompt; the new prompt applies at the next compaction (rebuilding that layer once) or in a new conversation.

### Cache scope

Effectively one machine and directory: the system prompt embeds auto-memory paths and the conversation opens with working directory, platform, shell and OS version. Parallel sessions in the same directory share a prefix; sequential ones only if the startup git status snapshot (branch, recent commits) matches. Agent SDK fleets can move the auto-memory location out of the system prompt to share caches across users and machines.

### Subagents and forks

- A subagent has its own system prompt and tools, so its first request doesn't read the parent's cache; it warms its own. The parent's cache is unaffected.
- A fork inherits system prompt, tools and history exactly and reads the parent's cache.
- Session copies get their isolation instruction appended at the end, so the original's cache stays valid.
- Resumed subagents can read the cache their original run warmed.
- Workflow fan-outs of same-prefix agents hold all but the first for up to 5 seconds so they read its cache.

### Where the cache lives, and gateways

- API key, Claude subscription, Claude Platform on AWS: Anthropic's infrastructure.
- Bedrock, Google Cloud's Agent Platform: the provider's infrastructure. Foundry: depends on whether the deployment is hosted on Azure or on Anthropic.
- Custom `ANTHROPIC_BASE_URL`, an LLM gateway, or a provider base-URL override (`ANTHROPIC_BEDROCK_BASE_URL`): depends on how the gateway treats `cache_control`:
  - forwards it unchanged: caches as at the provider;
  - rejects it with a 400 naming `cache_control`: Claude Code moves the marker to the last conversation message for the rest of the process; the mid-conversation system-context block bills uncached, the conversation stays cached;
  - strips it and returns success (or turns block-form system content into a plain string): **the entire history bills uncached every turn.** Check for this when costs look wrong behind a gateway.
- Claude Code marks its mid-conversation system-context block (file-change notices) for caching on every provider unless `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` is set.

### Cache lifetime in Claude Code and the Agent SDK

Every request falls in one of two buckets:
- **Main conversation**: interactive turns, `-p` runs, Agent SDK turns, and their inline helpers.
- **Everything else**: subagents, workflows, in-process teammates, forks, compaction, session titles.

| Bucket | Claude subscription within plan usage | Usage credits, API key, or cloud provider |
|---|---|---|
| Main conversation | 1 hour | 5 minutes |
| Everything else | 5 minutes (a few server-controlled helpers get 1 hour) | 5 minutes |

A subscription that goes over its limit and draws on usage credits drops the main conversation to 5 minutes unless you set the TTL yourself.

Controls (each takes `5m` or `1h`; other values are ignored; Claude Code v2.1.242+):
- Main conversation: setting `promptCacheTtl` or env `CLAUDE_CODE_PROMPT_CACHE_TTL`.
- Everything else: setting `subagentPromptCacheTtl` or env `CLAUDE_CODE_SUBAGENT_PROMPT_CACHE_TTL`.

Precedence, first match wins:
1. `FORCE_PROMPT_CACHING_5M=1` (5 minutes for both buckets; use when debugging, comparing TTLs, or overriding a managed 1h)
2. The bucket's environment variable
3. The bucket's setting
4. For a subagent, `cacheTtl` in its `experimental` frontmatter (v2.1.248+; a `1h` there is ignored while a subscription uses credits)
5. `ENABLE_PROMPT_CACHING_1H=1` (1 hour for both; older adapters may only read this)
6. The bucket default

With an API key or cloud provider, set `promptCacheTtl: "1h"` to give the main conversation an hour; other requests stay at 5 minutes until you set theirs. Verify with `claude -p "hello" --output-format json` and read `usage.cache_creation.ephemeral_1h_input_tokens` vs `ephemeral_5m_input_tokens`. Through a gateway set by `ANTHROPIC_BASE_URL`, the 1h request partly travels in the `anthropic-beta` header, so the gateway must forward it unchanged; a Claude apps gateway needs an admin policy; on Bedrock, support, minimums and 1h availability vary by model (zero cache counts usually mean unsupported).

Organization-wide policy: put TTL or disable variables in the `env` block of managed settings.

Disable caching (debugging only): `DISABLE_PROMPT_CACHING=1` (all models), or `DISABLE_PROMPT_CACHING_HAIKU`, `_SONNET`, `_OPUS` (each applies to the model its alias resolves to; a different pinned version of that family keeps caching), `DISABLE_PROMPT_CACHING_FABLE`.

---

## Apps built on the Agent SDK, Claude Code, or ACP

The SDK builds the requests, so the raw-API layout rules are handled. Your app still decides what goes *into* the session, and some of those choices still cost cache.

### What the SDK handles

- Prompt ordering and `cache_control` placement
- Injecting updates as messages instead of editing the system prompt
- Compaction that reuses the warm cache for the summary request
- Resuming a saved session: the resumed request reads whatever prefix is unchanged and still inside the TTL

### What your app still controls

| App action | Cost | Fix |
|---|---|---|
| **Changing** the model (`setModel()`, a model config option, a skill whose frontmatter names another model, `opusplan` toggling plan mode) | Each model has its own cache: the next request reads the whole history uncached | Lock the model per session; for cheaper side work, hand off to a subagent |
| **Changing** effort | Keeps the cache on Fable 5.1+, Opus 5.5, Sonnet 5.5 and Haiku 5.5 with an API key or Claude subscription. Invalidates on other models, on Bedrock, Google Cloud, gateways, or with experimental betas disabled | Fine on current models on the Anthropic API; elsewhere change it only at natural breaks |
| Toggling fast mode | Fast and standard speed keep separate caches | Choose it at the start of the session |
| **Re-sending** an unchanged model or effort every turn | Harmless for the cache in the SDK and the ACP adapter we checked, but it can override the user's own settings (see model-picker) | Send only on change |
| Changing which tools are defined (allowed tools, read-only modes that hide write tools, deny rules for a whole tool) | Invalidates the cache when tools load upfront | Register the same tools every turn; refuse disallowed calls when they run |
| Connecting or removing an MCP server mid-session | With tool search on, Claude Code keeps the first request's tool list, so it's kept. With tools loaded upfront, invalidated | Keep tool search on; keep servers registered and gate their calls |
| An MCP server whose `tools/list` depends on state, or that sends `notifications/tools/list_changed` | Same as adding or removing tools | Return a fixed list; move state into call arguments or results |
| Restarting the agent process and starting a **new** session | No cache: a new conversation | Reuse the process, or resume the old session |
| Restarting the process and **resuming** the session | Cached prefix is read if the resume happens inside the TTL and nothing in the prefix changed. Process startup and reconnects still cost time | Avoid restarts that aren't needed: key process reuse only on what truly needs a new process, and pass per-turn state (device, path, socket) as call arguments, not launch args or env |
| Accumulating many images | Claude Code drops the oldest in batches; each batch reprocesses from that point | Share images only when needed |
| Pauses longer than the TTL | Cache expires; next turn rebuilds it | Set `CLAUDE_CODE_PROMPT_CACHE_TTL=1h` (or the `promptCacheTtl` setting) for the main conversation; `CLAUDE_CODE_SUBAGENT_PROMPT_CACHE_TTL` for subagents. Subscriptions within plan usage already get 1h on the main conversation. Older adapters may read `ENABLE_PROMPT_CACHING_1H=1` instead, which has lower precedence |
| Routing through a gateway or custom base URL | A gateway that strips `cache_control` silently bills the whole history uncached every turn | Confirm cache read tokens are non-zero through the gateway; make it forward `cache_control` and the `anthropic-beta` header |

---

## Auditing

Goal: find everything that makes request N+1's prefix differ from request N's, and anything that throws away a warm cache. Report each break with file, line, estimated cost, and fix. Then confirm with numbers, not just code reading.

### Raw Messages API harness

1. Find where each request is assembled. List, in order, what goes into `tools`, `system`, and the first messages.
2. For every piece, ask: can it differ between two turns of one session? Look for timestamps, dates, random IDs, request IDs, user-specific data before shared content, counters, "current state" summaries, environment details that change, and maps serialized without a fixed order (tools, schemas, `tool_use` inputs).
3. Check the tool list: is it built from state (mode, permissions, connected servers, available agents)? Any branch that adds, removes or reorders tools, or edits a description or enum, is a full break. Fix with state tools plus messages, `defer_loading` with tool search, or inline `tool_addition`.
4. Check the system prompt: is it ever edited mid-session (mode switches, date, reminders)? Move those to mid-conversation system messages or a tagged block in the next user message / tool result.
5. Check history handling: does any code rewrite, trim, re-render or reorder earlier messages (truncating old tool results, re-rendering reminders with fresh values, dropping images, reformatting)? Each is a break from that point on.
6. Check model, effort, thinking, speed, `tool_choice`, web search and citations toggles: are any changed mid-session? Use per-message effort, a subagent hand-off, or fix them per session.
7. Check `cache_control`: is caching enabled at all? Is the breakpoint on the last stable block (not a per-request block)? Can a turn add 20+ blocks past the last write? Are there more than 4 breakpoints? Is the prefix above the minimum length?
8. Check side calls (compaction, summarization, classifiers, title generation): do they reuse the parent's exact tools, system and history? If not, they pay full price on what they send.
9. Check compaction: does it trigger with enough buffer, and run while warm rather than after an idle break?
10. Check timing: typical gaps vs TTL; concurrent requests that should share a prefix but are sent before the first response starts.
11. Check that `usage` cache fields are logged per request.

### Apps on the Agent SDK, Claude Code, or ACP

1. Find where the app creates or reuses the agent process or session, and list every input to that decision (the reuse key). For each, ask: does changing it start a new session, change the tool list, or change model, speed or effort? Each yes is a cost; report file and line.
2. Check how often each input changes. Anything regenerated per turn (a UUID, a temp path, a random token) that feeds the reuse key restarts the process every turn.
3. Check every MCP server the app ships: is `tools/list` identical on every call, and are JSON keys serialized in a stable order?
4. Check where the app changes model, effort, fast mode, allowed tools or permission rules mid-session, and compare against the table above.
5. Check the app keeps the cache numbers. Adapters report them in their own fields (the ACP adapter puts `cachedReadTokens` in its usage), not always as `cache_read_input_tokens`. If the app throws them away, nobody can see a cache break.
6. Confirm with those numbers: a mid-session turn with a large cache write and few cache reads marks the break.

### Example finding

> `AppModel.swift:412` — `readOnly` is part of the agent-process reuse key, and `AppMCPServer.swift:88` drops write tools when it's set, so toggling plan mode starts a new process with a different tool list: a full cache rebuild on every toggle. Fix: always advertise the write tools, refuse them in `tools/call` while read-only (the app already blocks them at `AppModel.swift:390`), and drop `readOnly` from the reuse key.

---

## Gotchas

- A precise timestamp (seconds or minutes) or today's date in the static system prompt. Put time in a message.
- Tools or JSON schemas serialized from unordered maps, so order or key order changes between requests (Swift and Go are common culprits, including keys inside `tool_use` blocks).
- Tool parameters or descriptions that depend on state, e.g. the list of agents a subagent tool can call.
- A breakpoint on a block that changes every request (it writes every time and never reads); automatic caching on a prompt whose last block varies has the same problem.
- A turn that adds 20+ blocks after the last write, pushing it out of the lookback window.
- Prefix below the model's minimum: silently uncached.
- Adding or removing images anywhere in the prompt (invalidates message blocks), including harness-side trimming of old images.
- Rewriting or trimming earlier messages, or editing/removing a mid-conversation system message after sending it. On models with preserved thinking (Fable 5.2 and others), altering content before a thinking block also gets the request rejected by the thinking prefix check.
- Per-request IDs, random seeds, or user-specific data placed before shared content.
- Changing `tool_choice`, toggling web search or citations, or flipping fast mode mid-session.
- Parallel requests fired before the first response started, so none of them hit.
- A separate summarization call with its own system prompt and no tools.
- Compacting after an idle break instead of before.
- Pre-warming with different thinking or effort settings than real traffic.
- A gateway that strips `cache_control` while returning success.
- Restarting the agent process because a per-turn value feeds the process reuse key.
- An SDK adapter renaming cache fields, and the app discarding them.

---

## Measure

### Usage fields

Every response's `usage` (or the `message_start` event when streaming) has:
- `cache_read_input_tokens`: read from cache this request.
- `cache_creation_input_tokens`: written to cache this request; equals the sum of `cache_creation.ephemeral_5m_input_tokens` and `cache_creation.ephemeral_1h_input_tokens`.
- `input_tokens`: only the tokens **after the last breakpoint**, not the total.

```
total_input = cache_read_input_tokens + cache_creation_input_tokens + input_tokens
hit_rate    = cache_read_input_tokens / total_input
```

Example: 100,000 read + 0 written + 50 fresh = 100,050 total input, hit rate ≈ 99.95%. In a healthy loop, each turn writes roughly the last assistant turn plus the new user turn and reads everything before it. A high read-to-creation ratio means caching works; creation staying high turn after turn means something in the prefix keeps changing. A large write with few reads in the middle of a session marks a break. Both read and write at 0 usually means the prefix is below the minimum (or caching isn't enabled).

Compaction requests report in `usage.iterations`; sum iterations for totals. SDK adapters may rename these fields (e.g. `cachedReadTokens`).

### Monitor like uptime

Track hit rate per session and in aggregate, alert on drops, and treat a cache break as an incident with a root cause, the way the Claude Code team does. A few percentage points of miss rate is a large change in cost and latency. When a break is unexplained, the API's cache diagnostics feature compares consecutive requests and reports where the prefix diverged. In Claude Code, the OpenTelemetry exporter reports cache read and creation tokens per user and session, and a status line script can read `current_usage` and the `prompt_cache` object live.

### Reading a Claude Code session

`/usage` (same as `/cost`) shows the session's input, output and cache figures, and a `Prompt cache (main)` line with the hit ratio, miss count, whether the cache is warm now, and often the likely cause of the last miss (e.g. `likely cause: tool definitions changed`). Three checks at the end of a task:

1. **Cache share.** It should be high on a long session. When it's low, check for a long pause, a model switch, an MCP server added mid-session, or (on a cloud provider or gateway) an effort change. Read the likely-cause text.
2. **Output against input.** Lots of output for a small change usually means effort is too high for the task, or the model is retrying.
3. **Total input against conversation size.** Many times the conversation's size means many turns, each resending the history; read the conversation to find where the loop repeated.

Why long sessions cost more per turn even with a perfect cache: every turn rereads the whole context. On Opus 5.5 a turn's cache read is about $0.004 at 20K tokens and $0.03 at 150K; 30 turns cost $0.12 vs $0.90 in reads. Bigger context windows don't change per-token price on Claude 4.6 and later, so the cost is entirely resending. Fewer turns (batched tool calls, a way for the model to check its own work), a lean CLAUDE.md (under about 200 lines, since every line is resent every turn), unused MCP servers turned off, `/clear` between unrelated tasks and `/compact` at breaks all reduce it.

---

## Lessons learned (from building Claude Code)

1. **Caching is a prefix match.** Changing anything in the prefix invalidates all that follows it. Design the whole system around this; get the ordering right and most caching is free.
2. **Use messages, not system prompt edits.** Plan mode, the date, changed files: insert them into the conversation instead of editing the system prompt.
3. **Don't change tools or models mid-conversation.** Model state transitions (like plan mode) as tools rather than by changing the tool set. Defer tool loading instead of removing tools. Switch models through a subagent hand-off.
4. **Monitor cache hit rate like uptime.** Alert on cache breaks and treat them as incidents; a few points of miss rate dramatically changes cost and latency.
5. **Fork operations share the parent's prefix.** Compaction, summarization, skill execution or any side computation should use identical cache-safe parameters (same tools, system, history, model, settings) so they hit the parent's cache.
6. **Build around caching from day one.** Claude Code was designed around prompt caching from the start, and the team recommends any agent be built the same way: decide modes, tools and compaction with the stable prefix in mind.

## Based on

- Thariq Shihipar, "Lessons from building Claude Code: Prompt caching is everything", claude.dev Blog (Anthropic), Apr 30, 2026. https://claude.dev/blog/lessons-from-building-claude-code-prompt-caching-is-everything/
- Addy Osmani, "What a task costs on Opus 5.5", claude.dev Blog (Anthropic), Sep 25, 2026. https://claude.dev/blog/what-a-task-costs-on-opus-5-5/
- "Prompt caching", Claude Platform docs (Anthropic). https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- "Mid-conversation system messages and tool changes", Claude Platform docs (Anthropic). https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages
- "Effort", Claude Platform docs (Anthropic). https://platform.claude.com/docs/en/build-with-claude/effort
- "Compaction" and "Compaction on demand", Claude Platform docs (Anthropic). https://platform.claude.com/docs/en/build-with-claude/compaction
- "Tool search tool", Claude Platform docs (Anthropic). https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool
- "How Claude Code uses prompt caching" and "Environment variables", Claude Code docs (Anthropic). https://code.claude.com/docs/en/prompt-caching
- "Tools: list changed notifications", Model Context Protocol specification. https://modelcontextprotocol.io/specification/2025-06-18/server/tools
