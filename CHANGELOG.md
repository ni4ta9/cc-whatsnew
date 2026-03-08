# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/), and this project adheres to [Semantic Versioning](https://semver.org/).

## [0.3.0] - 2026-03-08

### Added

- Template Customization Mode: `/whatsnew <design instruction>` generates a custom HTML template at `~/.claude/whatsnew-template.html` with interactive preview and refinement
- `/whatsnew reset` to delete the custom template and restore the default
- `Edit` added to allowed-tools to support interactive setup writes

## [0.2.1] - 2026-03-08

### Changed

- Convert SKILL.md to English-based templates with runtime translation via Language Policy
- Add HTML auto-open prompt after report generation

## [0.2.0] - 2026-03-08

### Added

- Interactive setup (Step 6): walk through recommended features and apply settings one by one
- Enhanced README with features list, how-it-works section, and screenshot

## [0.1.0] - 2026-03-07

### Added

- Initial release
- Personalized feature report generation (MD + HTML)
- User profile analysis via `/insights` integration
- Release notes fetching from Claude Code CHANGELOG
- Top Picks recommendation engine
- Latest version detailed breakdown
- Template-based HTML generation with JSON data injection
- Source attribution (copyright compliance)
