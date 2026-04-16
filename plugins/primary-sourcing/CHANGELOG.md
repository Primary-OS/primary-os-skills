# Changelog

All notable changes to the `primary-sourcing` plugin.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] — 2026-04-15

### Added
- Initial release of the team-wide `primary-sourcing` plugin.
- Four skills: `start-sourcing-project`, `kickoff-role`, `run-sourcing-batch`, `run-weekly-summary`.
- Six use cases: `recruiting-portco`, `gtm-sourcing-portco`, `investment-sourcing`, `fund-lp-sourcing`, `advisor-board-sourcing`, `other`.
- Bundled Project templates: `company-project/CLAUDE.md`, `role/SEARCH-TEMPLATE.md`, `role/config-template.json`.
- Comprehensive references for context intake, scoring rubric, dedup algorithm, Slack formatting, scheduled task prompts, collision handling, intro messages.
- Per-user Airtable schema: Searches, Candidates, Served Leads, Feedback.
- Dedup algorithm: per-user, same-search hard exclusion, max-one cross-search repeat per batch, burned-on-serve for repeats.

### Known limitations
- Requires the Lovelace MCP's Apify LinkedIn search tool, which is under development (tracked in Linear PRI-679).
- Requires the Lovelace Slack bot extensions for multi-user channel routing (tracked in Linear PRI-678).
- Live-brain updates from Slack feedback rely on the owner's Cowork scheduled tasks to apply changes (near-real-time, not truly live).

### Ported from
- Annabelle Kim's single-user `sourcing-agent` repo (Apify search, AI scoring, Airtable sync, Slack posting, weekly digest, live brain update). Original script paths documented in Linear PRI-678 and PRI-679 for provenance.
