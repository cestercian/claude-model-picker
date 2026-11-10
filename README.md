<div align="center">

# 🧠 claude-model-picker

**Stop guessing which Claude to use. Let the skill pick for you.**

Haiku 5.5 · Sonnet 5.5 · Opus 5.5 · Fable 5.2 — the right model *and* effort level, every time.

[![Agent Skill](https://img.shields.io/badge/Agent%20Skill-SKILL.md-d97757)](./skills/claude-model-picker/SKILL.md)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-plugin-black)](#-install-in-claude-code)
[![Sourced from Anthropic](https://img.shields.io/badge/sourced%20from-Anthropic%20docs-cc785c)](#-receipts-every-claim-is-cited)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](./LICENSE)

</div>

---

## ✨ Why this skill goes hard

Picking a model by vibes is lowkey burning your token budget. This skill takes Anthropic's own model-selection playbook and makes it something Claude can actually use mid-task:

- 🎯 **Task → model + effort, instantly.** Bug fix? Classification at scale? Massive migration? It already knows the starting point.
- 💸 **Cost per finished task, not per token.** A pricier model at lower effort can be the cheaper move, and this skill knows when.
- 🪜 **A no-cap escalation ladder.** Self-verify → raise effort → move up a model. In that order.
- 🧩 **Multi-model setups.** Cheap subagents, lead + worker swarms, advisor pairings, and when *not* to bother.
- 🛂 **Migration footguns flagged.** Preserved thinking 400s, `tool_choice` errors, Haiku 4.5 → 5.5 param changes, all of it.
- 🧾 **Receipts on every claim.** Linked straight back to the source docs below.

## 🧾 Receipts: every claim is cited

Everything in this skill comes straight from Anthropic's published guidance. Nothing is made up and nothing is vibes:

| Source | What it covers |
|---|---|
| 📝 [**Which Claude model should you build with? Fable, Opus, Sonnet, or Haiku**](https://claude.dev/blog/which-claude-model-should-you-build-with) — Addy Osmani, claude.dev Blog, Nov 9 2026 | The primary source: task table, trade-offs, mixing models, migration checklist |
| 📚 [Choosing a model](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model) — Claude Platform docs | Official selection guidance |
| 📊 [Models overview](https://platform.claude.com/docs/en/models/overview) — Claude Platform docs | Model IDs, pricing, context windows |
| 🎚️ [Effort](https://platform.claude.com/docs/en/build-with-claude/effort) — Claude Platform docs | Effort levels and trade-offs |
| ⚙️ [Model configuration](https://code.claude.com/docs/en/model-config) — Claude Code docs | Aliases, `opusplan`, env vars |
| 🤖 [Subagents](https://code.claude.com/docs/en/sub-agents) — Claude Code docs | Per-subagent `model:` settings |

> ⚠️ Model lineups and prices move fast. The skill tells Claude to check the live docs before relying on a specific price or model ID.

## 🚀 Install in Claude Code

Add this repo as a plugin marketplace:

```
/plugin marketplace add cestercian/claude-model-picker
```

Then install the plugin:

```
/plugin install claude-model-picker@claude-model-picker
```

### Or install it manually

```bash
git clone https://github.com/cestercian/claude-model-picker ~/.claude/skill-repos/claude-model-picker
```

```bash
ln -s ~/.claude/skill-repos/claude-model-picker/skills/claude-model-picker ~/.claude/skills/claude-model-picker
```

### Claude.ai and the API

Upload the [`skills/claude-model-picker`](./skills/claude-model-picker) folder as a custom skill. See [Using skills in Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude).

## 💬 Try it

Just ask Claude:

- *"Which model should I use to migrate this repo to TypeScript?"*
- *"My subagents are expensive. Make them cheaper."*
- *"We're moving our classifier from Haiku 4.5 to 5.5. What breaks?"*
- *"Opus at high still isn't cutting it. What next?"*

## 🗂️ Repo layout

```
.claude-plugin/
  marketplace.json          # Claude Code plugin marketplace manifest
skills/
  claude-model-picker/
    SKILL.md                # the skill: frontmatter + instructions + sources
LICENSE
README.md
```

Follows the layout of [anthropics/skills](https://github.com/anthropics/skills) and the [Agent Skills spec](https://agentskills.io).

## 📌 Disclaimer

This is a community skill. It is **not affiliated with or endorsed by Anthropic**. It summarizes Anthropic's public guidance in its own words and links to the originals. Claude, Haiku, Sonnet, Opus and Fable are trademarks of Anthropic. Always check the [official docs](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model) for the latest.

## 📄 License

[MIT](./LICENSE) for this repo's own wording and files. The guidance it summarizes belongs to Anthropic.
