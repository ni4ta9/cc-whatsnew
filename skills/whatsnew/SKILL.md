---
name: whatsnew
description: "Fetch Claude Code release notes and generate a personalized feature report (MD + HTML) based on your usage patterns. Use when asked about new features, latest updates, release notes, or feature recommendations. Invokable via /whatsnew."
allowed-tools: Bash, Read, Write, Skill, AskUserQuestion
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
awk '/__JSON_DATA__/{system("cat /tmp/whatsnew-data.json");next}1' \
  "${CLAUDE_SKILL_DIR}/assets/template.${LANG}.html" > "cc-whatsnew-${DATE}.html"
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
       open cc-whatsnew-<DATE>.html
```

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

`$ARGUMENTS` (optional) — No argument fetches the latest release notes. If a version is specified (e.g., `v0.2.0`), focus on information around that version.
