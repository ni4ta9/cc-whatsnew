# cc-whatsnew

> **Unofficial community plugin. Not affiliated with Anthropic.**
> Sources: [GitHub CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) · [Release Notes](https://docs.anthropic.com/en/docs/claude-code/release-notes)

A Claude Code plugin that fetches the latest release notes and generates a **personalized** MD + HTML report based on your usage patterns (skills, hooks, usage insights).

## Install

```bash
/plugin marketplace add ni4ta9/cc-whatsnew
/plugin install whatsnew@cc-whatsnew
```

## Usage

```
/whatsnew
```

Or in natural language:

- "What's new in Claude Code?"
- "Recommend features for me"
- 「Claude Code の新機能を教えて」

## Features

- Personalized top picks matched against your `/insights` usage profile
- Full release notes with categorized badges
- Standalone HTML report with Japanese-inspired design
- Multilingual: output language follows the `language` setting in `~/.claude/settings.json`

## License

MIT © 2026 ni4ta9
