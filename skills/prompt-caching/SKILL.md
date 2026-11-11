---
name: prompt-caching
description: Design or audit an agent so Claude prompt caching keeps hitting - raw Messages API harnesses and apps built on the Claude Agent SDK, Claude Code, or ACP. Use when building an agent loop, when Claude API costs or latency are higher than expected, when the cache hit rate is low, or when adding plan modes, model or effort switching, dynamic or MCP tools, timestamps, process restarts, or conversation compaction.
---

# Prompt caching for agents

## The one constraint

Caching is a **byte-for-byte prefix match** in the order `tools` → `system` → `messages`. Change one byte and everything after it is recomputed and re-billed. Agent loops resend everything every turn, so hit rate dominates input cost.

**Layout, most to least stable:** static system prompt + tools (global) → project context (CLAUDE.md, memory) → session context (cwd, platform, git snapshot) → messages, growing only at the end.

**How ordering breaks** (Claude Code hit all three): a timestamp in the static system prompt; non-deterministic tool order; tool parameters that change with state (e.g. callable agents). Each costs the whole conversation.

**API mechanics in brief:** caching is opt-in (top-level `"cache_control": {"type": "ephemeral"}` for automatic, or per-block breakpoints, max 4). A breakpoint writes one entry for the prefix ending there; reads look back up to 20 blocks for an entry an earlier request wrote. Put the breakpoint on the **last block identical across requests**; one on a per-request block writes every time and never reads.

## Who builds the requests?

- **Your code calls the Messages API:** everything here applies.
- **Your app drives the Agent SDK, Claude Code, or an ACP agent:** the SDK handles ordering, `cache_control`, updates as messages and cache-safe compaction. You still control model, effort, fast mode, tools, MCP servers, restarts, TTL and gateways (see the app table).

## Rules

1. **Put updates in messages, not the system prompt or tools.** Why: editing the system prompt invalidates everything after it; appending costs only new tokens.
   - Preferred: a mid-conversation system message, `{"role": "system", "content": "..."}` appended to `messages` (current 5.x models; not Sonnet 5). Keeps operator priority. Must follow a user turn; never between `tool_use` and `tool_result`.
   - Never edit or delete one after sending (it's history; also breaks later thinking blocks). To change an instruction, append a new one. Never put untrusted content (tool output, web pages) in one.
   - Any model: a `<system-reminder>` block in the next user message or tool result.
   - Per-loop nudges that shouldn't pile up: `"clear_at": "next_user_message"` (beta); resend verbatim forever, no `cache_control` on it.

2. **Don't switch models mid-session.** Why: caches are per model; the new model writes the whole history (1.25×) instead of reading it (0.1×). 100K tokens into Opus, an easy question is cheaper on Opus than Haiku. **Instead, hand off to a subagent:** the main model writes a short brief, a cheaper-model subagent starts small, and the parent's cache only sees an appended call and result. Disguised switches: `opusplan`, skill frontmatter `model`, safety fallback, fast mode on an unsupported model. If you must switch, compact first.

3. **Hold effort and thinking constant, or change them cache-safely.** Why: thinking config and top-level `output_config.effort` are rendered into the prompt; changing them invalidates messages (sometimes more). Setting effort to the model's default equals omitting it. On current 5.x models use per-message effort (beta): a `role: "system"` message with empty content and `"output_config": {"effort": "high"}`. Otherwise pick one level at session start. In Claude Code, effort changes keep the cache on current models via API key or subscription, but not on Bedrock, Google Cloud, gateways, or with experimental betas disabled.

4. **Never add or remove tools mid-session.** Why: tools come first; any add, remove, reorder or edit invalidates everything.
   - **Plan mode as tools:** keep every tool always; add `EnterPlanMode`/`ExitPlanMode`; on entry send a message with the rules; refuse disallowed calls at execution time. Same pattern for any mode.
   - **Defer, don't remove:** `defer_loading: true` plus a tool search tool (regex or BM25). Send every definition every request; keep the search tool and 3-5 common tools non-deferred and put the breakpoint on a non-deferred tool. Worth it at 10+ tools or several MCP servers.
   - **Inline `tool_addition` (beta):** append a system message carrying a new definition; leave `tools` as first sent. For tools unknown at start or whose schema changes.
   - **Stable serialization:** fixed tool order and JSON key order (Swift dictionaries and Go maps randomize), including `tool_use` inputs you send back. Move changing state out of descriptions/enums into messages.

5. **Forks share the parent's prefix.** Why: a side call (compaction, summary, review, skill run) with its own system prompt or tools shares nothing and pays full price. Copy the parent's last request and append the instruction.

6. **Pick the TTL for your idle gaps.** Why: after expiry the next turn pays a full write. Reused within 5 minutes: stay on 5m (hits refresh free, writes cheaper). Gaps of 5-60 minutes, or latency matters: use 1h.

## Compaction

**What:** when context fills, summarize and continue from the summary. It invalidates the message layer by design; what matters is the summary request's cost.

**The trap:** a separate "summarize this" call with its own system prompt and no tools diverges at the first token, so the whole conversation bills uncached, and costs more the longer it got.

**Cache-safe fork:**
1. Take the parent's exact last request: same tools, system, context blocks, model, thinking and effort.
2. Keep all messages unchanged.
3. Append one user message with the compaction instruction ("do not call tools", what to keep).
4. Send while warm; it reads the whole prefix from cache.
5. Replace history with the summary and continue.
6. **Compaction buffer:** trigger early enough that the window still fits the instruction plus summary output.

Compact before a break, not after (cold cache = full reprocessing). To abandon a wrong path, rewind instead (the remaining prefix is cached). For unrelated work, clear.

**API on-demand compaction (beta):** send the conversation with `"compaction": {"type": "summarize"}` and the same model, system, tools and thinking; put the returned `compaction` block first in `messages`, byte-for-byte, on every later request. Prefer it to your own summarizer.

## What invalidates / what keeps

| Change | Effect |
|---|---|
| Tool definitions added/removed/reordered/edited | Invalidates everything |
| Model switch | Whole history uncached (separate cache per model) |
| Effort / thinking params | Invalidates messages; Claude Code keeps it on current models via API key/subscription, per-message effort keeps it on the API |
| Fast mode toggle (incl. auto fallback and return) | Separate cache per speed; turn it on at session start |
| Web search/citations toggle; `tool_choice`, images added/removed | Invalidates system + messages; messages only |
| Editing or trimming earlier messages | Invalidates from that point |
| MCP connect/remove | Kept with tool search; invalidates with tools upfront (server dropping out or reconnecting: kept) |
| Deny rule on a whole tool | Kept with tool search; invalidates without. Scoped deny and allow/ask rules: kept |
| Claude Code dropping oldest images in batches | Reprocesses from the earliest dropped |
| Compaction | Invalidates message layer by design |
| Claude Code upgrade | First conversation after rebuilds from the top |
| Resume within TTL with unchanged prefix | Kept (resumed session keeps its original system prompt) |
| Appended system message, `<system-reminder>`, per-message effort, inline tool_addition | Kept |
| Repo or CLAUDE.md edits, permission mode, output style, skills, plugins, subagents, rewind | Kept |

## TTL essentials

- Default **5 minutes**, refreshed free on every hit, measured from the start of the request.
- 1h: `"ttl": "1h"`. Write cost: 5m = 1.25× input, 1h = 2×; read = 0.1× (lower on some current models).
- Claude Code: main conversation via `promptCacheTtl` or `CLAUDE_CODE_PROMPT_CACHE_TTL`; subagents, forks, compaction etc. via `subagentPromptCacheTtl` or `CLAUDE_CODE_SUBAGENT_PROMPT_CACHE_TTL` (values `5m`/`1h`). `FORCE_PROMPT_CACHING_5M=1` overrides; `ENABLE_PROMPT_CACHING_1H=1` is the older, lower-precedence switch.
- Defaults: subscription within plan = 1h main, 5m other; API key, credits, cloud = 5m.

## Apps on the Agent SDK, Claude Code, or ACP

| App action | Cost | Fix |
|---|---|---|
| Changing model (`setModel()`, skill `model`, `opusplan`) | Whole history uncached | Lock per session; subagent hand-off |
| Changing effort | Kept on current models via Anthropic API; else invalidates | Elsewhere, change only at breaks |
| Toggling fast mode | Separate caches | Choose at session start |
| Re-sending unchanged model/effort | Harmless to cache; may override user settings | Send only on change |
| Changing allowed tools / read-only tool hiding | Invalidates with upfront tools | Same tools always; refuse at call time |
| MCP connect/remove; stateful `tools/list` or `list_changed` | Invalidates with upfront tools | Keep tool search on; fixed tool list |
| Restart process + new session | No cache | Reuse process or resume |
| Restart + resume within TTL | Cache read; startup still costs | Don't key process reuse on per-turn values |
| Pauses over TTL | Full rebuild | 1h TTL |
| Gateway / custom base URL | Stripping `cache_control` bills all history uncached | Check cache reads are non-zero; forward `cache_control` and `anthropic-beta` |

## Auditing

Find whatever makes request N+1's prefix differ from N's or discards a warm cache. Report file, line, cost, fix; confirm with numbers.

1. Find where requests (or the agent process/session) are built. List what goes into tools, system, first messages, and the process reuse key.
2. Can any piece change between turns? Timestamps, IDs, temp paths, user data before shared content, unordered maps.
3. Tool list or MCP `tools/list` built from state (mode, permissions, servers, agents)? Full break.
4. System prompt edited mid-session? Move to messages.
5. Earlier messages rewritten, trimmed, re-rendered, images dropped?
6. Model, effort, thinking, speed, `tool_choice`, web search/citations changed mid-session?
7. `cache_control` enabled, on the last stable block, ≤4 breakpoints, under 20 blocks per turn?
8. Side calls and compaction reuse the parent's exact prefix, with buffer, while warm?
9. Idle gaps vs TTL.
10. Cache fields logged per request (adapters may rename them, e.g. ACP `cachedReadTokens`)? A mid-session large write with few reads marks the break.

Example finding:
> `AppModel.swift:412` — `readOnly` is in the process reuse key and `AppMCPServer.swift:88` drops write tools when set, so each plan-mode toggle starts a new process with a different tool list: full rebuild. Fix: always advertise write tools, refuse them in `tools/call` while read-only, drop `readOnly` from the key.

## Gotchas

- Timestamp or date in the static system prompt.
- Unordered-map serialization; state in tool descriptions.
- Breakpoint on a per-request block; automatic caching when the last block varies.
- A turn adding 20+ blocks past the last write.
- Prefix below the model minimum: silently uncached.
- Adding, removing or trimming images; editing earlier or sent system messages.
- Toggling `tool_choice`, web search, citations, fast mode.
- Parallel requests sent before the first response starts streaming.
- Separate summarization call with its own prompt.
- Compacting after an idle break.
- Gateway stripping `cache_control` while returning success.
- Per-turn values in the process reuse key; discarded cache fields.

## Measure

`usage` fields: `cache_read_input_tokens`, `cache_creation_input_tokens` (5m + 1h split under `cache_creation`), `input_tokens` (only tokens after the last breakpoint).

```
total_input = cache_read_input_tokens + cache_creation_input_tokens + input_tokens
hit_rate    = cache_read_input_tokens / total_input
```

Healthy: each turn writes only the newest turn. Creation high every turn = prefix changing. Both zero = caching off or below minimum.

Claude Code `/usage` (shows a `Prompt cache (main)` line with likely cause of the last miss). Three checks:
1. **Cache share** should be high; if low, look for a pause, model switch, added MCP server, or effort change on a cloud provider.
2. **Output vs input:** heavy output for a small change = effort too high or retries.
3. **Total input vs conversation size:** a large multiple means many turns.

## Lessons learned (from building Claude Code)

1. **Caching is a prefix match.** Get ordering right and most caching is free.
2. **Use messages, not system prompt edits** for plan mode, dates, changed files.
3. **Don't change tools or models mid-conversation.** Model modes as tools, defer instead of removing, switch models via subagent hand-off.
4. **Monitor hit rate like uptime.** Treat breaks as incidents.
5. **Forks share the parent's prefix.** Compaction, summaries and side computations use identical tools, system, history, model and settings.
6. **Build around caching from day one.** Decide modes, tools and compaction with the stable prefix in mind.

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
