---
name: prompt-caching
description: Design or audit an agent so Claude prompt caching keeps hitting - raw Messages API harnesses and apps built on the Claude Agent SDK, Claude Code, or ACP. Use when building an agent loop, when Claude API costs or latency are higher than expected, when the cache hit rate is low, or when adding plan modes, model or effort switching, dynamic or MCP tools, timestamps, process restarts, or conversation compaction.
---

# Prompt caching for agents

Prompt caching is a **byte-for-byte prefix match** in the order `tools` → `system` → `messages`. Any change in the prefix invalidates everything after it. Design around that one constraint; get the ordering right and most caching works for free.

## First: who builds the requests?

- **Your code calls the Messages API** → everything below applies.
- **Your app drives the Agent SDK, Claude Code, or an ACP agent** → the SDK already orders the prompt, places `cache_control`, injects reminders as messages, and compacts safely. Skip those parts. What your app still controls, and can still break, is in [references/sdk-apps.md](references/sdk-apps.md). Read it before auditing.

## Auditing

Trace how each request (or each SDK session) is assembled turn by turn, and find anything that makes request N+1's prefix differ from request N's. Report each break with file, line, cost, and fix. Then confirm with numbers, not just code reading: see "Measure".

## Request layout (raw API)

Most stable first: static system prompt and tools → project context (e.g. CLAUDE.md) → session context → conversation messages.

## Rules

- **Put updates in messages, not the system prompt.** New time, changed files, mode switches: on current models append a mid-conversation `{"role": "system"}` message instead of editing top-level `system` (not available on Sonnet 5). Otherwise put the update in the next user message or tool result; Claude Code uses a `<system-reminder>` tag.
- **Don't switch models mid-session.** Caches are per model. 100K tokens into an Opus session, an easy question is cheaper on Opus than on Haiku, because Haiku must rebuild the whole cache. To use a cheaper model, have the main model write a short hand-off and give it to a subagent.
- **Don't change effort or thinking settings mid-session by editing the request.** Changing `output_config.effort` invalidates message blocks. Setting effort explicitly to the model's default is the same as omitting it. On models that support it, per-message effort (beta) changes effort and keeps the cache.
- **Never add or remove tools mid-session.** Tool definitions are the first thing in the prefix; changing them invalidates everything. Instead:
  - Model state changes as tools: plan mode is an `EnterPlanMode`/`ExitPlanMode` pair plus a message with the mode's rules, not a swap to a read-only tool set. Refuse disallowed calls at execution time.
  - Defer tools: send stubs with `defer_loading: true` in a fixed order and let the model load full schemas via tool search.
  - Add tools inline: with the `inline-tools-2026-09-15` beta, send a `tool_addition` in a mid-conversation system message and leave `tools` untouched.
- **Compact without breaking the cache.** Prefer the API's [on-demand compaction](https://platform.claude.com/docs/en/build-with-claude/compaction) when your app needs to control when compaction runs, can't pause for a summary, or must keep recent turns word for word. If you write your own, use a cache-safe fork: same system prompt, context and tools as the parent, the parent's messages, then the compaction instruction as a new user message. A separate "summarize this" call with its own system prompt and no tools misses the cache on the entire conversation. Reserve context-window room for the instruction and the summary.
- **Pick the TTL for your idle gaps.** The cache lives 5 minutes by default and refreshes free on each hit. 1-hour TTL costs 2× base input on writes (5-minute writes cost 1.25×). If users or agents routinely pause longer than 5 minutes, every resume pays a full write.

## Gotchas

- A precise timestamp (seconds or minutes) in the static system prompt.
- Tools or JSON schemas serialized from unordered maps, so order or key order changes between requests.
- Tool parameters that depend on state, e.g. the list of agents a subagent tool can call.
- Adding or removing images anywhere in the prompt (invalidates message blocks).
- Rewriting or trimming earlier messages (also rejected when it alters content before a Fable 5.2 thinking block).
- Per-request IDs, random seeds, or user-specific data placed before shared content.

## Measure

Every API response's `usage` has `cache_read_input_tokens`, `cache_creation_input_tokens`, and `input_tokens`; their sum is total input. Hit rate = reads ÷ total input. Alert on hit-rate drops like on uptime. SDK adapters may rename these fields; see [references/sdk-apps.md](references/sdk-apps.md).

In Claude Code, `/usage` (or `/cost`) shows the same for a session. Three checks at the end of a task:

1. **Cache share.** It should be high on a long session. If it's low, look for a long pause, a model switch, an MCP server connected partway through, or (on a cloud provider or gateway) an effort change. The prompt-cache line often names the likely cause of the last miss.
2. **Output against input.** Lots of output for a small change usually means effort is too high for the task, or the model is retrying.
3. **Total input against conversation size.** Many times the conversation's size means many turns, each resending the history.

## Sources

- Thariq Shihipar, ["Lessons from building Claude Code: Prompt caching is everything"](https://claude.dev/blog/lessons-from-building-claude-code-prompt-caching-is-everything/), claude.dev Blog (Anthropic), Apr 30, 2026.
- [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) (TTLs, what invalidates the cache, usage fields), [Mid-conversation system messages and tool changes](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages), [Effort](https://platform.claude.com/docs/en/build-with-claude/effort), [Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction) and [Tool search](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool), Claude Platform docs.
- [How Claude Code uses prompt caching](https://code.claude.com/docs/en/prompt-caching) and [Environment variables](https://code.claude.com/docs/en/env-vars), Claude Code docs.
- Addy Osmani, ["What a task costs on Opus 5.5"](https://claude.dev/blog/what-a-task-costs-on-opus-5-5/), claude.dev Blog (Anthropic), Sep 25, 2026 (reading a session).
- [Tools: list changed notifications](https://modelcontextprotocol.io/specification/2025-06-18/server/tools), Model Context Protocol spec.
