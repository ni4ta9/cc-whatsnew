# whatsnew — Claude Code Personalized What's New

> **Unofficial community plugin. Not affiliated with Anthropic.**
> Sources: [GitHub CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) · [Release Notes](https://docs.anthropic.com/en/docs/claude-code/release-notes)

Fetches the latest Claude Code release notes and generates a **personalized** MD + HTML report based on your usage patterns (skills, hooks, usage insights).

---

## Usage

```
/whatsnew
```

Or trigger naturally:

- "What's new in Claude Code?"
- "Show me the latest updates"
- "Recommend features for me"
- 「Claude Code の新機能を教えて」
- 「自分の使い方に合った機能を紹介して」

## Installation

### Option A: Plugin install (recommended)

```bash
/plugin marketplace add ni4ta9/cc-whatsnew
/plugin install whatsnew@cc-whatsnew
```

### Option B: Manual install

```bash
git clone --depth 1 https://github.com/ni4ta9/cc-whatsnew /tmp/cc-whatsnew
cp -r /tmp/cc-whatsnew/skills/whatsnew ~/.claude/skills/whatsnew
```

Then reload: `/reload-plugins`

## How it works

Uses built-in Claude Code commands internally:

| Step | Built-in used | Purpose |
|------|--------------|---------|
| Step 1 | `/insights` | Reads your usage profile from `~/.claude/usage-data/report.html` |
| Step 2 | — | Fetches CHANGELOG via `curl` (GitHub raw) |
| Step 3–5 | — | Matches features to your profile → generates a personalized MD + HTML report tailored to how you actually use Claude Code |

> **Note**: `/insights` must have been run at least once to generate a usage profile.
> If no profile exists, the skill will prompt you to run `/insights` first.

## Output

| File | Description |
|------|-------------|
| `cc-whatsnew-YYYY-MM-DD.md` | Markdown report (current directory) |
| `cc-whatsnew-YYYY-MM-DD.html` | Standalone HTML report (current directory) |

## Multilingual

The report language is determined by the `language` field in `~/.claude/settings.json`.

| `settings.json` value | Output language |
|-----------------------|-----------------|
| `"Japanese"`          | Japanese        |
| `"English"` or unset  | English         |
| Other values          | That language   |

## Template Customization

```bash
# Copy and customize the HTML template (language-specific)
cp ~/.claude/skills/whatsnew/assets/template.ja.html ~/.claude/whatsnew-template.html
```

| Priority | Path | Description |
|----------|------|-------------|
| 1 (high) | `~/.claude/whatsnew-template.html` | User custom template |
| 2 (low)  | `assets/template.{lang}.html` (bundled) | Default template (ja/en) |

To revert to default: `rm ~/.claude/whatsnew-template.html`

## Directory Structure

```
skills/whatsnew/
├── SKILL.md              # Skill definition (read by Claude)
├── README.md             # This file
├── LICENSE               # MIT License
└── assets/
    ├── template.ja.md    # Japanese MD template
    ├── template.en.md    # English MD template
    ├── template.ja.html  # Japanese HTML template
    └── template.en.html  # English HTML template
```

## License

MIT © 2026 ni4ta9

Release notes data sourced from [anthropics/claude-code](https://github.com/anthropics/claude-code) (public repository).
