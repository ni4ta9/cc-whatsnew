# cc-whatsnew

> **Unofficial community plugin. Not affiliated with Anthropic.**
> Sources: [GitHub CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) · [Release Notes](https://docs.anthropic.com/en/docs/claude-code/release-notes)

A Claude Code plugin that fetches the latest release notes and generates a **personalized** feature report based on your actual usage patterns.

<p align="center">
  <img src="assets/screenshot.png" alt="cc-whatsnew HTML report" width="600">
</p>

## Quick Start

```bash
# Install
/plugin marketplace add ni4ta9/cc-whatsnew
/plugin install whatsnew@cc-whatsnew

# Run
/whatsnew
```

Or in natural language:

- "What's new in Claude Code?"
- "Recommend features for me"
- "Claude Code の新機能を教えて"

## Features

- **Personalized Picks** — Top feature recommendations matched against your `/insights` usage profile
- **Categorized Release Notes** — Full changelog with visual badges (New / Improved / Fix / Breaking)
- **Standalone HTML Report** — Beautiful single-file report with Japanese-inspired design, no dependencies
- **Multilingual** — Output language follows your `settings.json` language preference
- **Template Customizable** — Generate a custom HTML template with `/whatsnew <design instruction>`, or edit manually
- **Interactive Setup** — Optionally apply recommended features to `CLAUDE.md` / `settings.json` one by one after report generation

## How it works

| Step | What happens |
|------|-------------|
| 1 | Reads your usage profile via `/insights` |
| 2 | Fetches the latest CHANGELOG from GitHub |
| 3 | Matches features to your profile and ranks by relevance |
| 4 | Generates a personalized MD + HTML report |
| 5 | Asks whether to open the HTML report in your browser |
| 6 | (Optional) Interactively applies recommended features to `CLAUDE.md` / `settings.json` |

> `/insights` must have been run at least once. If no profile exists, the skill will prompt you to run it first.

## Output

Reports are generated in the current directory:

| File | Description |
|------|-------------|
| `cc-whatsnew-YYYY-MM-DD.md` | Markdown report |
| `cc-whatsnew-YYYY-MM-DD.html` | Standalone HTML report (open in browser) |

## Multilingual

The report language is determined by the `language` field in `~/.claude/settings.json`.

| `settings.json` value | Output language |
|-----------------------|-----------------|
| `"Japanese"`          | Japanese        |
| `"English"` or unset  | English         |
| Other values          | That language   |

## Template Customization

You can customize the HTML report template in two ways:

**Option A — Generate with a command** (recommended)

```
/whatsnew make it dark
/whatsnew minimal and compact
/whatsnew washi paper texture style
```

Claude generates a custom `~/.claude/whatsnew-template.html` based on your design instruction, shows a preview, and lets you refine it interactively.

To revert to default: `/whatsnew reset` (or `rm ~/.claude/whatsnew-template.html`)

**Option B — Edit manually**

```bash
cp ~/.claude/skills/whatsnew/assets/template.ja.html ~/.claude/whatsnew-template.html
# Edit to your liking
```

See [skills/whatsnew/README.md](skills/whatsnew/README.md) for full template priority details.

## Manual Install

If you prefer not to use the plugin marketplace:

```bash
git clone --depth 1 https://github.com/ni4ta9/cc-whatsnew /tmp/cc-whatsnew
cp -r /tmp/cc-whatsnew/skills/whatsnew ~/.claude/skills/whatsnew
```

Then reload: `/reload-plugins`

## Links

- X: [#ccwhatsnew](https://x.com/hashtag/ccwhatsnew)
- Issues: [GitHub Issues](https://github.com/ni4ta9/cc-whatsnew/issues)
- Detailed docs: [skills/whatsnew/README.md](skills/whatsnew/README.md)

## License

MIT © 2026 ni4ta9
