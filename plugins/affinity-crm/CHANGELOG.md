# Changelog

## 0.2.0 — 2026-04-16

### Changed
- **BREAKING:** Replaced four skills (`manage-deal-pipeline`, `research-entities`, `manage-notes-and-activities`, `manage-lists-and-fields`) with one consolidated skill **`affinity-expert`** for clearer triggering and lower skill-budget use. Reference markdown files are unchanged in content and now live under `skills/affinity-expert/references/`.
- Added [`MCP-WORKER-PATCHES.md`](./MCP-WORKER-PATCHES.md) — spec for the Cloudflare Worker (server instructions, tool-description hints, optional `get_affinity_help`).

## 0.1.0 — 2026-04-16

### Added
- Initial release with four skills: `manage-deal-pipeline`, `research-entities`, `manage-notes-and-activities`, `manage-lists-and-fields`.
- Comprehensive reference documentation for all 8 custom tools and ~24 proxied upstream tools.
- Pipeline workflow guides covering logging, status updates, ownership, and reporting.
- Entity research strategies: name search, domain search, semantic search, pipeline enrichment, relationship intelligence.
- Notes and activity management: creation, entity linking, meeting history, reminders, transcripts.
- List and field operations: navigation, field value reads/writes, dropdown management, saved views.
