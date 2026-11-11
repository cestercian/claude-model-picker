# Caching in apps built on the Agent SDK, Claude Code, or ACP

The SDK builds the requests, so the raw-API layout rules are handled. Your app still decides what goes *into* the session, and some of those choices still cost cache. Anthropic's [How Claude Code uses prompt caching](https://code.claude.com/docs/en/prompt-caching) is the reference for what does and doesn't invalidate; read it when a case below is unclear.

## What the SDK handles

- Prompt ordering and `cache_control` placement
- Injecting updates as messages instead of editing the system prompt
- Compaction that reuses the warm cache for the summary request
- Resuming a saved session: the resumed request reads whatever prefix is unchanged and still inside the TTL

## What your app still controls

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

## How to audit

1. Find where the app creates or reuses the agent process or session, and list every input to that decision (the reuse key). For each, ask: does changing it start a new session, change the tool list, or change model, speed or effort? Each yes is a cost; report file and line.
2. Check how often each input changes. Anything regenerated per turn (a UUID, a temp path, a random token) that feeds the reuse key restarts the process every turn.
3. Check every MCP server the app ships: is `tools/list` identical on every call, and are JSON keys serialized in a stable order?
4. Check the app keeps the cache numbers. Adapters report them in their own fields (the ACP adapter puts `cachedReadTokens` in its usage), not always as `cache_read_input_tokens`. If the app throws them away, nobody can see a cache break.
5. Confirm with those numbers: a mid-session turn with a large cache write and few cache reads marks the break.

## Example finding

> `AppModel.swift:412` — `readOnly` is part of the agent-process reuse key, and `AppMCPServer.swift:88` drops write tools when it's set, so toggling plan mode starts a new process with a different tool list: a full cache rebuild on every toggle. Fix: always advertise the write tools, refuse them in `tools/call` while read-only (the app already blocks them at `AppModel.swift:390`), and drop `readOnly` from the reuse key.
