# Caching in apps built on the Agent SDK, Claude Code, or ACP

The SDK builds the requests, so the raw-API layout rules are handled. Your app still decides what goes *into* the session, and changing any of these mid-conversation still breaks the cache.

## What the SDK handles

- Prompt ordering and `cache_control` placement
- Injecting updates as messages instead of editing the system prompt
- Compaction that reuses the cached prefix
- Resuming a saved session

## What your app still controls

| App action | Why it breaks the cache | Fix |
|---|---|---|
| Changing the tool set between turns (`allowedTools`, `mcpServers`, read-only modes that hide write tools) | Tools are the first part of the prefix | Register the same tools every turn; refuse disallowed calls when they run |
| Adding or removing a whole MCP server on a toggle | Every tool from that server enters or leaves the prefix | Keep the server registered; gate its calls |
| An MCP server whose `tools/list` depends on state, or that emits `notifications/tools/list_changed` mid-session | The SDK re-reads the tool list and the definitions change | Return a fixed list; move state into call arguments or results |
| Baking per-turn state (device, scheme, path, flags) into the MCP server's launch args or env, when a change restarts the process | A restart rebuilds the session's environment and tool context | Pass that state per call (a tool argument or a broker the tool reads) |
| Restarting the agent process instead of reusing it | A fresh process starts a new session unless you resume it | Reuse the process; key reuse only on what truly needs a new session; resume (not new session) after a crash |
| Calling `setModel()` or changing effort every turn | A different model has no cache; changing effort invalidates message blocks | Lock model per session (hand off to a subagent for cheaper work); only send effort when it actually changes, and don't send the default as if it were a change |
| Waiting more than 5 minutes between turns | The default cache TTL expires | Accept it, or use a 1-hour TTL where the SDK exposes it |

## How to audit

1. Find where the app creates or reuses the agent process or session, and list every input to that decision (the reuse key).
2. For each input, ask: does changing it restart the process, change the tool list, or change the model or effort? Each yes is a cache break; report its file and line.
3. Check every MCP server the app ships: is `tools/list` the same every call, and are its JSON keys serialized in a stable order?
4. Confirm with the `usage` numbers the SDK reports per turn: a turn with high `cache_creation_input_tokens` and low `cache_read_input_tokens` mid-session points to the break.

## Example finding

> `AppModel.swift:412` — `readOnly` is part of the agent-process reuse key, and `AppMCPServer.swift:88` drops write tools when it's set, so toggling plan mode restarts the process with a different tool list: a full cache rebuild on every toggle. Fix: always advertise the write tools, refuse them in `tools/call` while read-only (the app already blocks them at `AppModel.swift:390`), and drop `readOnly` from the reuse key.
