---
name: whatsnew
description: "Fetch Claude Code release notes and generate a personalized feature report (MD + HTML) based on your usage patterns. Use when asked about new features, latest updates, release notes, or feature recommendations. Invokable via /whatsnew."
allowed-tools: Bash, Read, Write, Edit, Skill, AskUserQuestion
---

# What's New — Claude Code Personalized Report

Fetch the latest Claude Code release notes, analyze your usage patterns, and highlight the features most relevant to you. Output as MD + HTML reports.

## Output Files

- `cc-whatsnew-<YYYY-MM-DD>.md` — Markdown report (current directory)
- `cc-whatsnew-<YYYY-MM-DD>.html` — Standalone HTML report (current directory)

---

## Restrictions

- **Do NOT use WebFetch/Fetch tools** — Use Bash curl only for CHANGELOG retrieval
- **Do NOT use Glob/Search/Grep tools** — Use Bash `test -f` for file checks, Read for content
- **Do NOT run ls or check existing files on your own** — Use AskUserQuestion for overwrite confirmation
- **Do NOT write HTML files directly with Write tool** — HTML is generated in Step 4 via Bash using JSON + template

## Cases Requiring Confirmation

- **report.html exists**: Ask user whether to use existing or run `/insights`
- **report.html does not exist**: Ask user whether to run `/insights` or fall back to `settings.json` (no auto-execution)
- **Output files already exist**: Ask user whether to overwrite or cancel (for both MD and HTML)

## Processing Policy

- Execute sub-steps in parallel where dependencies allow
- Trust Bash command output; do not run additional Search/Grep/ls for verification
- Extract needed information directly from Read output; do not re-search

## Language Policy

All user-facing messages (AskUserQuestion prompts, choices, completion reports, setup dialogs) MUST be displayed in the language determined in Step 1a. The messages in this SKILL.md are written in English as templates — translate them to the target language at runtime. Do NOT display English messages to non-English users.

---

## Template Customization Mode

> **Entry condition**: `$ARGUMENTS` is non-empty and does NOT look like a version number.

If this mode is triggered, **do not run Steps 1–6**. Run Step T1–T5 instead.

### Step T1: Detect Mode and Handle `reset`

Check `$ARGUMENTS`:

- If `reset` (case-insensitive): Run `rm -f ~/.claude/whatsnew-template.html` via `Bash`, then report:
  > "Custom template deleted. The default template will be used from now on."
  > Terminate.
- Otherwise: Proceed to Step T2.

### Step T2: Load Reference Template

Run the following in parallel:

1. Check `~/.claude/whatsnew-template.html` via `Bash` (`test -f ... && echo EXISTS || echo NOT_FOUND`)
2. Determine language from Step 1a logic (read `~/.claude/settings.json`)

Then:

- If custom template EXISTS: Read `~/.claude/whatsnew-template.html` as the base template
- If NOT EXISTS: Read `${CLAUDE_SKILL_DIR}/assets/template.${LANG}.html` as the base template

### Step T3: Generate Custom Template

Generate a **complete, standalone HTML file** based on the user's design instruction in `$ARGUMENTS`.

#### Required Constraints (MUST follow exactly)

**1. JSON data injection placeholder**

The template MUST contain this exact block:

```html
<script id="whatsnew-data" type="application/json">
__JSON_DATA__
</script>
```

The `__JSON_DATA__` placeholder is replaced at runtime by `awk`. Do NOT hardcode data here.

**2. JSON data schema**

The JavaScript renderer must handle this JSON structure (injected at `__JSON_DATA__`):

```json
{
  "date": "YYYY-MM-DD",
  "latest_version": "vX.Y.Z",
  "picks_title": "あなたへのおすすめ — TOP N",
  "profile_summary": "Profile summary sentence.",
  "profile_tags": ["tag1", "tag2"],
  "picks": [
    {"rank": 1, "title": "Feature name", "reason": "Reason text", "howto": "Command example"}
  ],
  "details": [
    {"title": "Feature name", "overview": "One sentence.", "merit": "2–3 sentences.", "tip": "Optional command or null"}
  ],
  "releases": [
    {
      "version": "vX.Y.Z",
      "date": "YYYY-MM",
      "items": [{"badge": "feat|fix|improve|exp", "text": "Description"}]
    }
  ]
}
```

**3. Required HTML elements** (used by the JS renderer)

The template MUST include elements that the JavaScript can target. You may use any CSS selectors, but the JS must correctly reference what you define:

| Data | Suggested selector | Content |
|---|---|---|
| Header eyebrow | `.header-eyebrow` | `"Claude Code — " + D.date` |
| Profile summary | `.profile-summary` | `D.profile_summary` |
| Profile tags | `.tag-list` | `D.profile_tags.map(...)` |
| Picks section title | `#top-picks .section-title` | `D.picks_title` |
| Picks cards | `.picks-grid` | `D.picks.map(...)` |
| Detail section title | `#latest-detail .section-title` | `D.latest_version + " — ..."` |
| Detail cards | `.detail-grid` | `D.details.map(...)` |
| Releases container | `#releases-container` | `D.releases.map(...)` |
| Footer left | `.footer-left` | `"generated by /whatsnew · " + D.date` |
| Footer credit | `.footer-credit` | Link to `https://github.com/ni4ta9/cc-whatsnew` |

**4. Required JavaScript helpers**

Include these utility functions in the `<script>` block:

```javascript
function esc(s) {
  return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;')
    .replace(/>/g,'&gt;').replace(/"/g,'&quot;').replace(/'/g,'&#39;');
}
function md(s) {
  return esc(s)
    .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
    .replace(/`(.+?)`/g, '<code>$1</code>');
}
var VALID_BADGES = {feat:1, fix:1, improve:1, exp:1, other:1};
function safeBadge(b) { return VALID_BADGES[b] ? b : 'other'; }
```

**5. Required badge styles**

The template must style these badge classes: `.badge.feat`, `.badge.fix`, `.badge.improve`, `.badge.exp`, `.badge.other`

**6. Security**

Include this CSP meta tag:

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'none'; style-src 'unsafe-inline'; script-src 'unsafe-inline'">
```

#### Creative Freedom

Beyond the required constraints above, apply the user's design instruction freely:

- Color scheme, typography, background textures
- Layout (grid, cards, table, etc.)
- Animations, transitions
- Font choices (web-safe or Google Fonts via `<link>`)
- Section order, card shapes, badge styles
- Dark/light/sepia/custom themes

#### Writing the Template

Write the generated template to `~/.claude/whatsnew-template.html` using the `Write` tool.

### Step T4: Preview with Sample Data

After writing the template, generate a preview HTML with sample data to let the user visually verify.

Write the following sample JSON to `/tmp/whatsnew-preview.json` using `Write`:

```json
{
  "date": "2026-01-01",
  "latest_version": "v2.1.71",
  "picks_title": "Top Picks for You — TOP 3",
  "profile_summary": "Power user actively building custom skills. Heavy Bash usage with polyglot development and daily Git workflows.",
  "profile_tags": ["plugin-dev", "bash-heavy", "git-workflow", "polyglot"],
  "picks": [
    {"rank": 1, "title": "/loop command", "reason": "Directly useful for heavy custom-skill users. **Automates** repetitive tasks one level up.", "howto": "/loop 5m /commit"},
    {"rank": 2, "title": "Bash auto-approval expansion", "reason": "`test` and `printf` are now **auto-approved**. Fewer approval dialogs.", "howto": "Auto-applied from v2.1.71, no config needed"},
    {"rank": 3, "title": "/debug mid-session toggle", "reason": "Toggle debug logging on/off without restarting the session.", "howto": "/debug"}
  ],
  "details": [
    {"title": "/loop command + cron scheduling", "overview": "Run a prompt or slash command repeatedly at a set interval.", "merit": "Active within the session, ideal for deploy monitoring and periodic commit checks. **Heavy custom-skill users** can automate repetitive tasks with ease.", "tip": "/loop 5m check the deploy"},
    {"title": "Bash auto-approval allowlist expansion", "overview": "`fmt`, `test`, `printf` and more added to auto-approval.", "merit": "Significantly fewer approval dialogs for heavy Bash users. Applied immediately, no configuration required.", "tip": null},
    {"title": "/debug mid-session toggle", "overview": "Toggle debug logs without restarting the session.", "merit": "Enable logging only when needed and reduce noise during normal operation.", "tip": null}
  ],
  "releases": [
    {
      "version": "v2.1.71", "date": "2026-01",
      "items": [
        {"badge": "feat", "text": "Add `/loop` command (repeat at intervals)"},
        {"badge": "feat", "text": "Expand bash auto-approval: `fmt`, `test`, `printf`, etc."},
        {"badge": "fix", "text": "Fix keystop freeze in long-running sessions"},
        {"badge": "improve", "text": "`/plugin uninstall` now modifies settings.local.json"}
      ]
    },
    {
      "version": "v2.1.70", "date": "2025-12",
      "items": [
        {"badge": "fix", "text": "Fix clipboard corruption for CJK and emoji text"},
        {"badge": "improve", "text": "Reduce prompt re-renders by ~74%"},
        {"badge": "feat", "text": "[VSCode] Add native MCP server management dialog"}
      ]
    },
    {
      "version": "v2.1.69", "date": "2025-11",
      "items": [
        {"badge": "feat", "text": "Add `${CLAUDE_SKILL_DIR}` variable (skills can reference their own directory)"},
        {"badge": "feat", "text": "Add 10 languages to Voice STT (20 total)"},
        {"badge": "fix", "text": "Fix symlink bypass allowing writes outside working directory"},
        {"badge": "exp", "text": "Add `InstructionsLoaded` hook event"}
      ]
    }
  ]
}
```

Then generate the preview HTML via `Bash`:

```bash
awk '/__JSON_DATA__/{system("cat /tmp/whatsnew-preview.json");next}1' \
  ~/.claude/whatsnew-template.html > /tmp/whatsnew-preview.html
```

Then clean up and ask the user:

```bash
rm -f /tmp/whatsnew-preview.json
```

### Step T5: Confirm and Complete

Report what was changed:

```
Template saved.

  Location: ~/.claude/whatsnew-template.html
  Design: <1–2 sentences describing the applied design>
```

Ask with `AskUserQuestion` (**no skipping**):

> "Open the preview in your browser?"
> Choices: `["Open", "Close"]`

- "Open": Run `open /tmp/whatsnew-preview.html`
- "Close": Run `rm -f /tmp/whatsnew-preview.html`

Then ask:

> "Would you like to adjust the template further?"
> Choices: `["Adjust further", "Use as-is"]`

- "Adjust further": Ask `AskUserQuestion` with question `"How would you like to change it?"` (free text), then re-run Step T3 with the new instruction, keeping the current template as the base.
- "Use as-is": Report `"Done. Your custom template will be used the next time you run /whatsnew."` and terminate.

---

## Processing Steps

Execute the following steps according to their dependencies.

### Step 1: Initial Data Collection

Execute the following sub-steps in parallel.

#### 1a: Determine Output Language

Read `~/.claude/settings.json` with the `Read` tool and determine the output language from the `language` field. All subsequent steps (report body, HTML labels, footer text, AskUserQuestion messages) MUST use this language consistently.

| `language` value | Output language | `lang` code |
|----------------|---------|--------------|
| `"Japanese"` | Japanese | `ja` |
| `"English"` or unset | English | `en` |
| Other | Corresponding language | BCP47 code |

#### 1b: Collect User Profile (Insight)

Run `test -f ~/.claude/usage-data/report.html && echo "EXISTS" || echo "NOT_FOUND"` via `Bash`.

**If report exists**: Read the first 50 lines with `Read` to check the date range in `<p class="subtitle">`. Then **always** ask with `AskUserQuestion` (no auto-selection, no skipping):

> "An Insight report for [date range] exists at `~/.claude/usage-data/report.html`. How would you like to proceed?"
> Choices: `["Use existing report (recommended)", "Run /insights fresh"]`

- "Use existing report": Proceed to the next Read
- "Run /insights fresh": Invoke the `insights` skill via `Skill` tool

**If report does not exist**: **Always** ask with `AskUserQuestion` (no auto-selection, no skipping):

> "`report.html` not found. `/insights` is needed to generate it. How would you like to proceed?"
> Choices: `["Run /insights (recommended)", "Use settings.json as fallback (less accurate)"]`

- "Run /insights": Invoke the `insights` skill via `Skill` tool
- "Use settings.json as fallback": Estimate profile from `~/.claude/settings.json`, skill list, and hook configuration

**In either case**: Read `~/.claude/usage-data/report.html` once with the `Read` tool and extract the following:

| Information | HTML hint |
|---|---|
| Session count, message count, active days | `stat-value` class values |
| Main goals/task types | `What You Wanted` section |
| Frequently used tools | `Top Tools Used` section |
| Primary languages | `Languages` section |
| Active projects | `project-area` section |
| Friction points | `friction-category` section |
| Underutilized feature suggestions | `feature-card` section |
| Success patterns | `big-win` section |
| Overall summary | `at-a-glance` section |

Internally note the collected information as a bullet-point profile. Example:

- Backend-heavy workload
- High security awareness (multiple security hooks)
- Active in automation/workflow development
- Actively creating custom skills
- Friction point: tends to forget running tests after file edits

#### 1c: Fetch Release Notes

> **Do NOT use WebFetch/Fetch tools. Always use `Bash` + `curl`.**

<!-- NOTE: /release-notes is a built-in Claude Code command but not a prompt-based skill,
     so invoking it via Skill tool causes "not a prompt-based skill" error.
     WebFetch loads the full 129KB into context and slows down, so use Bash curl + awk to extract only the first 3 versions. -->

Run the following via `Bash` to get only the first 3 versions from the CHANGELOG:

```bash
# NOTE: Never use the -k flag (disable TLS verification)
curl -s https://raw.githubusercontent.com/anthropics/claude-code/refs/heads/main/CHANGELOG.md \
  | awk '/^## /{count++} count>3{exit} {print}'
```

Extract the following from the output:

- Release content for each version/date
- Feature categories (new features / improvements / bug fixes / experimental)
- Target audience (developers / CI-CD / enterprise, etc.)

#### 1d: Load Templates and Check Output Files

> Execute after 1a (language determination) completes. Can run in parallel with 1b and 1c.

**Template loading:**

Select templates based on the language code determined in 1a:

| Language | MD template | HTML template |
|---|---|---|
| `ja` | `${CLAUDE_SKILL_DIR}/assets/template.ja.md` | `${CLAUDE_SKILL_DIR}/assets/template.ja.html` |
| `en` | `${CLAUDE_SKILL_DIR}/assets/template.en.md` | `${CLAUDE_SKILL_DIR}/assets/template.en.html` |
| Other | `${CLAUDE_SKILL_DIR}/assets/template.en.md` (fallback) | `${CLAUDE_SKILL_DIR}/assets/template.en.html` (fallback) |

If a user-custom HTML template exists at `~/.claude/whatsnew-template.html`, use it instead (check with `Bash` `test -f`).

Read the selected templates in parallel with `Read`.

**Output file check:**

- Run `test -f cc-whatsnew-<DATE>.md && echo "EXISTS" || echo "NOT_FOUND"` via `Bash`
- Run `test -f cc-whatsnew-<DATE>.html && echo "EXISTS" || echo "NOT_FOUND"` via `Bash`

If either file already exists, ask with `AskUserQuestion`:

> "The following files already exist: `cc-whatsnew-<DATE>.md` / `cc-whatsnew-<DATE>.html`. Overwrite?"
> Choices: `["Overwrite", "Cancel"]`

If cancelled, terminate the skill.

If overwriting, delete existing files via `Bash` (to make Step 4's Write a fresh create for speed):

```bash
rm -f cc-whatsnew-<DATE>.md cc-whatsnew-<DATE>.html
```

---

### Step 2: Content Analysis

After Step 1 completes, execute the following sub-steps in parallel.

#### 2a: Generate Personalized Recommendations

Cross-reference the profile from 1b with the feature list from 1c to determine "Top Picks for You".

**Matching logic (highest priority first):**

1. **Direct match**: Features directly related to skills/tools the user uses
   - Example: Heavy external service integration → prioritize MCP/tool integration features
   - Example: Frequent commit/security skills → prioritize Git integration / security enhancements

2. **Workflow complement**: Features that can enhance the current workflow
   - Example: Many custom skills → prioritize skill creation/management features
   - Example: Heavy hook usage → prioritize hook/automation features

3. **Underutilized features**: Features the user likely hasn't used yet but would find useful
   - Suggest when no corresponding tool is found in the profile

For each recommended feature, include:

- Why this feature is relevant to you (1-2 sentences, specific)
- First step to get started (command or configuration example)

#### 2b: Generate Latest Version Detail

From the release notes fetched in 1c, **explain each new feature (Added) of the latest version (top entry) in detail**.

For each feature, summarize:

- **Overview** (1 sentence): What you can now do
- **Benefits** (2-3 sentences): Why you should use it, how it specifically helps
- **Usage tips** (if applicable): Command or configuration examples

Granularity guidelines:

- Simple UI improvements (spinner display, etc.): 1-2 sentences, concise
- Hooks/skills/settings affecting development workflow: 3-4 sentences, thorough
- Security fixes: Clearly state what was patched and impact on users

---

### Step 3: Generate Output Files

After Step 2 completes, execute 3a and 3b **in parallel in the same turn**.

#### 3a: Write MD File

Replace placeholders in the MD template loaded in 1d and `Write`:

| Placeholder | Content | Format |
|---|---|---|
| `__DATE__` | Today's date | YYYY-MM-DD |
| `__LATEST_VERSION__` | Latest version number | As-is |
| `__SECTION_PICKS_TITLE__` | `TOP N` (N = number of recommendations) | As-is |
| `__TOP_PICKS_MD__` | Recommended features (2a results) | `### 1. Title` + reason + getting started |
| `__LATEST_DETAIL_MD__` | Latest version details (2b results) | Heading + body per feature |
| `__ALL_FEATURES_MD__` | All release notes (1c data) | `### vX.Y.Z` per version |
| `__PROFILE_MD__` | Profile summary + tags | Bullet list |

#### 3b: Write JSON Data File

Write data collected in Steps 1-2 to `/tmp/whatsnew-data.json` following this JSON schema. The JavaScript in the HTML template reads this JSON to render the page.

```json
{
  "date": "YYYY-MM-DD",
  "latest_version": "vX.Y.Z",
  "picks_title": "Top Picks for You — TOP N",
  "profile_summary": "Profile summary (1-2 sentences)",
  "profile_tags": ["tag1", "tag2"],
  "picks": [
    {"rank": 1, "title": "Feature name", "reason": "Reason", "howto": "Command example"}
  ],
  "details": [
    {"title": "Feature name", "overview": "Overview", "merit": "Benefits", "tip": "Tip (optional)"}
  ],
  "releases": [
    {
      "version": "vX.Y.Z",
      "date": "YYYY-MM-DD",
      "items": [{"badge": "feat", "text": "Description"}]
    }
  ]
}
```

**badge values**: `feat` (new feature), `fix` (fix), `improve` (improvement), `exp` (experimental)

**Emphasis in text**: Use `**text**` for bold, `` `code` `` for inline code (the HTML renderer will convert them)

---

### Step 4: Generate HTML

Inject the JSON file from Step 3b into the HTML template loaded in 1d. Run the following via `Bash`:

```bash
# Use custom template if it exists, otherwise use the default language template
if test -f ~/.claude/whatsnew-template.html; then
  HTML_TEMPLATE=~/.claude/whatsnew-template.html
else
  HTML_TEMPLATE="${CLAUDE_SKILL_DIR}/assets/template.${LANG}.html"
fi
awk '/__JSON_DATA__/{system("cat /tmp/whatsnew-data.json");next}1' \
  "$HTML_TEMPLATE" > "cc-whatsnew-${DATE}.html"
```

> The JavaScript in the HTML template renders page content from the JSON data. The LLM does not need to generate HTML tags directly.

---

### Step 5: Completion Report + Setup Proposal

Delete temporary files:

```bash
rm -f /tmp/whatsnew-data.json
```

Display the completion message, then **immediately in the same turn** call `AskUserQuestion`. Do NOT end after just displaying the report.

> **This Step is NOT complete until `AskUserQuestion` is called. Do not skip.**

Message to display:

```
Done! Personalized report generated.

  Your profile: <2-3 key characteristics>
  Recommended features: <N>
  Release notes entries: <N>

  MD : cc-whatsnew-<DATE>.md
  HTML: cc-whatsnew-<DATE>.html
```

Then ask with `AskUserQuestion` (**no skipping, no auto-selection**):

> "Open the HTML report in your browser?"
> Choices: `["Open", "No thanks"]`

- "Open": Run `open cc-whatsnew-<DATE>.html` via `Bash`
- "No thanks": Do nothing and proceed

Then ask with `AskUserQuestion` (**no skipping, no auto-selection**):

> "Would you like to apply the recommended feature settings now? We can walk through each one and update your CLAUDE.md or settings.json."
> Choices: `["Apply (one by one)", "Skip"]`

- "Apply": Proceed to Step 6
- "Skip": Terminate the skill

---

### Step 6: Interactive Setup (Optional)

Apply each "Top Pick" feature from Step 2a interactively, in rank order.

#### Processing Loop

For each feature (rank 1 → 2 → ... → N), repeat the following:

**1. Display feature description:**

```
-- Recommendation <rank>/<N>: <feature name> --

<Overview: what it does (2-3 sentences)>

How to set up:
  <Target file and specific changes>
```

Determine the target based on feature type:

| Feature type | Target | Example change |
|---|---|---|
| Hook (PreToolUse / PostToolUse, etc.) | `~/.claude/settings.json` `hooks` | Add hook entry |
| Allowed command | `~/.claude/settings.json` `allowedTools` | Add tool name |
| Setting value (model, language, etc.) | `~/.claude/settings.json` | Set key and value |
| Workflow/operational practice | Project `CLAUDE.md` | Append instructions/rules |
| Skill usage suggestion | None (explanation only) | No config change needed |

**2. Ask user (`AskUserQuestion`):**

> "Apply this setting?"
> Choices: `["Apply", "Skip", "Stop here"]`

**3. Process based on selection:**

- **"Apply"**: Read the target file with `Read`, then apply changes with `Edit` or `Write`. Report the change in one line.
- **"Skip"**: Do nothing and move to the next feature.
- **"Stop here"**: Break the loop and proceed to the completion summary.

#### Notes on Applying

- When editing `settings.json`, read current content with `Read` and use `Edit` for partial updates to avoid breaking existing structure
- When appending to `CLAUDE.md`, add a section at the end (do not modify existing content)
- If an equivalent setting already exists, skip and report "Already configured"
- **For features that don't require config changes** (skill usage suggestions, etc.), display explanation only and skip the apply confirmation

#### Completion Summary

After the loop ends (all items processed or "Stop here" selected), display a summary:

```
-- Setup Complete --

  Applied: <applied feature names, comma-separated>
  Skipped: <skipped feature names, comma-separated>
  Modified files: <list of changed file paths>
```

Then ask with `AskUserQuestion` (**no skipping, no auto-selection**):

> "Would you like to apply the recommended feature settings now? We can walk through each one and update your CLAUDE.md or settings.json."
> Choices: `["Apply (one by one)", "Skip"]`

- "Apply": Proceed to Step 6
- "Skip": Terminate the skill

---

## Source Attribution (Copyright Compliance)

All output generated by this skill (MD and HTML) MUST include source attribution. Do not omit.

### Markdown Report Footer (Required)

```markdown
---
*Sources: [GitHub CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) · [Release Notes](https://docs.anthropic.com/en/docs/claude-code/release-notes)*
*Generated by /whatsnew (https://github.com/ni4ta9/cc-whatsnew) — Unofficial community plugin, not affiliated with Anthropic.*
```

### HTML Report Footer

The footer (official release notes link, credits, source attribution) is hardcoded in the HTML template and rendered by JS automatically. The LLM does not need to include footer-related fields in the JSON data.

---

## Arguments

`$ARGUMENTS` (optional):

| Pattern | Behavior |
|---|---|
| *(empty)* | Fetch latest release notes and generate a personalized report |
| Version number (e.g., `v0.2.0`, `2.1.71`) | Focus report on that version |
| Design instruction (e.g., `make it dark`, `minimal and compact`) | Enter **Template Customization Mode** — generate a custom `~/.claude/whatsnew-template.html` |

**Template Customization Mode** is triggered when `$ARGUMENTS` is non-empty and does NOT look like a version number. Examples:

- `/whatsnew make it dark`
- `/whatsnew make it minimal and compact`
- `/whatsnew washi paper texture style`
- `/whatsnew reset` → delete `~/.claude/whatsnew-template.html` and restore default
