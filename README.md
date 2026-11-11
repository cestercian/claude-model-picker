# claude-playbook-skills

Agent skills based on Anthropic's published guidance for building with Claude. Each skill lists its sources at the bottom of its `SKILL.md`.

| Skill | What it does | Source |
|---|---|---|
| [model-picker](./skills/model-picker) | Choose a Claude model and effort level for a task, subagent or API integration | [Which Claude model should you build with?](https://claude.dev/blog/which-claude-model-should-you-build-with) |
| [prompt-caching](./skills/prompt-caching) | Design or audit an agent harness so prompt caching keeps hitting | [Prompt caching is everything](https://claude.dev/blog/lessons-from-building-claude-code-prompt-caching-is-everything/) |
| [code-review-setup](./skills/code-review-setup) | Set up layered, risk-tiered code review with AI and security review on every PR | [How the Claude Code team does code review](https://claude.dev/blog/solving-code-review-at-anthropic/) |

## Install

In Claude Code:

```
/plugin marketplace add cestercian/claude-playbook-skills
/plugin install model-picker@claude-playbook-skills
/plugin install prompt-caching@claude-playbook-skills
/plugin install code-review-setup@claude-playbook-skills
```

For Claude.ai or the API, upload a skill's folder as a custom skill.

## Disclaimer

Not affiliated with Anthropic. Check the [official docs](https://platform.claude.com/docs) for current models, pricing and features.

## License

MIT
