# Affinda Skills

Agent skills for integrating with [Affinda's](https://www.affinda.com) document AI platform.

Skills follow the [Agent Skills](https://agentskills.io/specification) open standard and work with Claude Code, OpenAI Codex, GitHub Copilot, and other compatible agents.

## Available Skills

| Skill | Description |
|-------|-------------|
| [`affinda`](skills/affinda/SKILL.md) | Complete guide to integrating with the Affinda document processing API — authentication, client libraries, structured outputs, webhooks, and the full documentation map. |

## Installation

```bash
# Claude Code
claude skills add affinda/skills

# Or manually copy skills/affinda/ into your project's .claude/skills/ directory
```

## Links

- [Affinda Documentation](https://docs.affinda.com)
- [Affinda API Reference](https://docs.affinda.com/reference/getting-started)
- [OpenAPI Spec](https://api.affinda.com/static/v3/api_spec.yaml)
- [Python Client Library](https://github.com/affinda/affinda-python)
- [TypeScript Client Library](https://github.com/affinda/affinda-typescript)
