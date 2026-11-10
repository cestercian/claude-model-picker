# claude-model-picker

A skill that helps Claude choose a model (Haiku 5.5, Sonnet 5.5, Opus 5.5, Fable 5.2) and effort level for a task, subagent, or API integration.

Based on Anthropic's [Which Claude model should you build with?](https://claude.dev/blog/which-claude-model-should-you-build-with) and the [Choosing a model](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model) docs. Full sources are listed in [SKILL.md](./skills/claude-model-picker/SKILL.md#sources).

## Install

In Claude Code:

```
/plugin marketplace add cestercian/claude-model-picker
/plugin install claude-model-picker@claude-model-picker
```

For Claude.ai or the API, upload the [`skills/claude-model-picker`](./skills/claude-model-picker) folder as a custom skill.

## Disclaimer

Not affiliated with Anthropic. Check the [official docs](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model) for current models and pricing.

## License

MIT
