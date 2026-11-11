---
name: prompt-caching
description: Design or audit an agent harness built on the Claude API so prompt caching keeps hitting. Use when building an agent loop, when Claude API costs or latency are higher than expected, when the cache hit rate is low, or when adding plan modes, model switching, dynamic tools, timestamps, or conversation compaction to an agent.
---

# Prompt caching for agent harnesses

Claude API prompt caching is a **byte-for-byte prefix match** up to each `cache_control` breakpoint. Any change in the prefix invalidates everything after it. Design the harness around that one constraint; get the ordering right and most caching works for free.

When auditing code, trace how each request is assembled turn by turn and look for anything that makes request N+1's prefix differ from request N's. Report each cache break with file, line, and the fix.

## Request layout

Order content from most stable to least stable:

1. Static system prompt and tool definitions (shared across all users/sessions)
2. Project-level context, e.g. CLAUDE.md (shared within a project)
3. Session context (shared within a session)
4. Conversation messages (grow each turn)

## Rules

- **Put updates in messages, not the system prompt.** New time, changed files, mode switches: append them to the next user message or tool result (Claude Code uses a `<system-reminder>` tag) instead of editing earlier content.
- **Don't switch models mid-session.** Caches are per model. 100K tokens into an Opus session, an easy question is cheaper on Opus than on Haiku, because Haiku must rebuild the whole cache. To use a cheaper model, have the main model write a short hand-off and give it to a subagent.
- **Never add or remove tools mid-session.** Tools are part of the prefix. Model state changes as tools instead: plan mode is an `EnterPlanMode`/`ExitPlanMode` tool pair plus a message with the mode's rules, not a swap to a read-only tool set.
- **Defer tools instead of removing them.** Send lightweight stubs with `defer_loading: true` in a fixed order and let the model load full schemas via tool search when needed.
- **Compact with a cache-safe fork.** Don't summarize with a separate call that has its own system prompt and no tools; that misses the cache on the entire conversation, and gets more expensive the longer the conversation. Reuse the parent's exact system prompt, context, and tools, keep the parent's messages, and append the compaction instruction as a new user message. Reserve a buffer in the context window for that instruction and the summary output. The API also has built-in compaction.
- **Monitor cache hit rate like uptime.** Alert on drops and treat cache breaks as incidents.

## Gotchas

Common ways teams have broken the prefix without noticing:

- A precise timestamp (down to seconds or minutes) in the static system prompt.
- Tool definitions built from an unordered map or set, so their order changes between requests.
- Tool parameters that depend on state, e.g. the list of agents a sub-agent tool can call changing mid-session.
- Rewriting or trimming earlier messages (also rejected outright when it alters content before a Fable 5.2 thinking block).
- Per-request IDs, random seeds, or user-specific data placed before shared content.

## Sources

- Thariq Shihipar, ["Lessons from building Claude Code: Prompt caching is everything"](https://claude.dev/blog/lessons-from-building-claude-code-prompt-caching-is-everything/), claude.dev Blog (Anthropic), Apr 30, 2026.
- [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) and [Tool search](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool), Claude Platform docs.
