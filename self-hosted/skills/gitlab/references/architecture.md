# GitLab Architecture & Database Schema Reference

Reference for understanding what's under the hood of a self-hosted GitLab instance — data stores, PostgreSQL schema design, table organization, and internal patterns.

## Data Stores

A self-hosted GitLab instance relies on these data stores:

| Store | Type | Role |
|---|---|---|
| **PostgreSQL** | Relational DB | Primary persistent store — all application metadata: users, projects, issues, MRs, CI pipelines, permissions, settings |
| **Redis** | In-memory | Session data, caching, Sidekiq background job queues (~25 KB per user) |
| **Gitaly** | Custom RPC service | Stores and serves bare Git repositories on disk — commits, blobs, branches, tags, hooks |
| **Object Storage** | S3-compatible | CI artifacts, LFS objects, user uploads (avatars, attachments), container registry images, GitLab Pages content |
| **Elasticsearch** | Search engine | Advanced code/issue search (optional, EE feature) |
| **Prometheus** | Time-series DB | Metrics collection for monitoring the instance itself |

### Supporting infrastructure

| Component | Role |
|---|---|
| **PgBouncer** | PostgreSQL connection pooling (used in larger deployments) |
| **Consul** | Service discovery and database node failover coordination (EE) |
| **Redis Sentinel** | Redis high availability |
| **Patroni** | PostgreSQL leader selection and replication (HA setups) |
| **Praefect** | Gitaly proxy for coordinating replication across multiple Gitaly nodes |

### What lives where

| Data | Store |
|---|---|
| User accounts, permissions, settings | PostgreSQL |
| Projects, groups, namespaces | PostgreSQL |
| Issues, MRs, comments, labels, milestones | PostgreSQL |
| CI pipeline configs, job metadata, runner registration | PostgreSQL |
| Audit logs, abuse reports | PostgreSQL |
| Git repository data (commits, trees, blobs) | Gitaly (disk) |
| CI build artifacts, job logs | Object Storage (or local disk) |
| LFS objects, user file uploads | Object Storage (or local disk) |
| Container registry images | Object Storage |
| User sessions, page caches | Redis |
| Background job queues (Sidekiq) | Redis |
| Rate limiting counters | Redis |
| Metrics and performance data | Prometheus |

All connections between components use Unix sockets by default, with TCP available for distributed/Kubernetes deployments.

## PostgreSQL Schema Overview

GitLab defines its schema in `db/structure.sql` (plain SQL, not Rails `schema.rb`) to leverage PostgreSQL-specific features like triggers, partitioning, and materialized views. The schema is ~34,000 lines with **~573 tables**.

### Database Decomposition

GitLab maintains up to 5 logical databases:

| Database | Schema | Contents |
|---|---|---|
| **main** | `gitlab_main` | Core app data — `projects`, `users`, `namespaces`, `issues`, `merge_requests`, `notes`, `milestones`, etc. |
| **ci** | `gitlab_ci` | CI/CD data — `ci_builds`, `ci_pipelines`, `ci_runners`, `ci_stages`, `ci_variables`, etc. |
| **sec** | `gitlab_sec` | Security scan results |
| **embedding** | — | AI/embedding features (EE only) |
| **geo** | — | Disaster recovery / geo-replication metadata (EE only) |

For small self-hosted instances these all run in a single PostgreSQL instance. GitLab.com runs main and ci as physically separate databases. CI tables account for ~40-50% of all write traffic, which motivated the split.

### Cross-database communication

- **Mirrored tables**: `ci_namespace_mirrors` and `ci_project_mirrors` are replicated from main to ci via logical replication so CI can resolve project/namespace without cross-DB joins.
- **Loose foreign keys**: PostgreSQL triggers + async workers handle cascading deletes across the database boundary.
- **Safety check**: GitLab raises a cross-database modification error when a single transaction tries to modify tables in different databases.

## Key Tables & Relationships

### Core entities

| Table | Purpose | Notes |
|---|---|---|
| `users` | User accounts | |
| `namespaces` | Groups and user namespaces | Hierarchical via `parent_id` |
| `projects` | Repositories | Belong to a namespace |
| `members` | User-to-namespace/project membership | Stores role/access level |

### Code review

| Table | Purpose |
|---|---|
| `merge_requests` | Merge requests, linked to `projects` |
| `merge_request_diffs` | Diff data for each MR version |
| `merge_request_diff_commits` | Individual commits in MR diffs |
| `merge_request_metrics` | Performance metrics per MR |
| `approvals` | MR approvals |

### Issues & planning

| Table | Purpose |
|---|---|
| `issues` | Issues, linked to `projects` |
| `labels` | Label definitions |
| `label_links` | Polymorphic join table linking labels to issues/MRs |
| `milestones` | Milestone definitions |
| `boards` | Issue boards |
| `epics` | Epics (EE) |

### Notes & activity

| Table | Purpose |
|---|---|
| `notes` | Comments on issues, MRs, snippets, commits (polymorphic via `noteable_type`) |
| `events` | Activity feed events |
| `award_emoji` | Emoji reactions |
| `todos` | User to-do items |

### CI/CD (in the `ci` database)

| Table | Purpose |
|---|---|
| `ci_pipelines` | Pipeline runs |
| `ci_builds` | Individual jobs (one of the largest tables) |
| `ci_stages` | Pipeline stages |
| `ci_runners` | Registered runners |
| `ci_variables` | CI/CD variables |
| `ci_trigger_requests` | Pipeline trigger data |
| `ci_namespace_mirrors` | Replicated from main DB for cross-DB lookups |
| `ci_project_mirrors` | Replicated from main DB for cross-DB lookups |

### Security & compliance

| Table | Purpose | Notes |
|---|---|---|
| `audit_events` | Audit log entries | Range-partitioned by time, no FKs (append-only) |
| `abuse_reports` | Abuse reports | No FKs |
| `web_hook_logs` | Webhook delivery logs | Range-partitioned, no FKs |
| `spam_logs` | Spam detection logs | No FKs |

### Infrastructure & deployment

| Table | Purpose |
|---|---|
| `deployments` | Deployment records |
| `environments` | Deployment environments |
| `deploy_tokens` | Deploy token credentials |
| `clusters` | Kubernetes cluster integrations |
| `pages_domains` | GitLab Pages custom domains |

### Auth & access

| Table | Purpose |
|---|---|
| `oauth_applications` | OAuth app registrations |
| `personal_access_tokens` | PATs |
| `deploy_keys` | SSH deploy keys |
| `keys` | User SSH keys |
| `authentication_events` | Login/auth events |

## Schema Design Patterns

### Primary key strategy

| Type | Count | Notes |
|---|---|---|
| `bigserial` (8-byte) | ~380 tables | Preferred for new tables |
| `serial` (4-byte) | ~170 tables | Legacy, potential scalability risk at scale |
| Composite PKs | ~23 tables | Join/mapping tables |
| UUID | 0 tables | Not used |

### Dual ID pattern

Tables like `issues`, `ci_pipelines`, `deployments`, and `epics` use two identifiers:
- **`id`**: Internal primary key (never exposed to users)
- **`iid`**: User-facing identifier scoped to the parent project/namespace (e.g., Issue #42)

This prevents ID guessing and provides better UX by resetting counters per project.

### Data types

| Pattern | Type Used | Example |
|---|---|---|
| Text fields | `text` with `CHECK` constraints | Preferred over `varchar(n)` for schema evolution flexibility |
| System timestamps | `timestamp without time zone` | `created_at`, `updated_at` |
| User-facing timestamps | `timestamp with time zone` | Scheduled events, user actions |
| IP addresses | `inet` | `audit_events`, `authentication_events` |
| Hashes/tokens | `bytea` | SHA hashes, encrypted tokens, fingerprints |
| Flexible metadata | `jsonb` | Request payloads, parameters, settings |
| Enums | `smallint` | Space-efficient over character types |
| Related ID lists | `integer[]` / `bigint[]` | `mentioned_users_ids`, `tag_ids` |

### Naming conventions

- Tables: **plural**, **snake_case** (`issues`, `merge_requests`)
- Module prefixes group related tables: `merge_request_*`, `ci_*`, `project_*`
- Booleans: `is_xxx`, `has_xxx`, or feature-flag style (`confidential`, `discussion_locked`)
- Indexes: `index_#{table}_on_#{columns}_#{condition}`

### Foreign keys

Used in most tables with strategic exceptions:
- **`ON DELETE CASCADE`**: Automatic child deletion (most common)
- **`ON DELETE RESTRICT`**: Prevents parent deletion if children exist
- **`ON DELETE SET NULL`**: Orphans rows with NULL references
- **No FK constraints**: `audit_events`, `abuse_reports`, `web_hooks_logs`, `spam_logs` (immutable, high-volume append-only tables)

### Partitioning

| Strategy | Tables | Use case |
|---|---|---|
| RANGE | `audit_events`, `web_hook_logs` | Time-series data |
| LIST | `loose_foreign_keys_deleted_records` | Discrete value distribution |
| HASH | `product_analytics_events_experimental` | Even distribution |

### Search indexes

- **GIN trigram indexes** (`gin_trgm_ops`) for `LIKE`/`ILIKE` searches on text fields
- **`tsvector`** for full-text search
- Extensive use of partial indexes with `WHERE` conditions

### Other patterns

- **Optimistic locking**: `lock_version` column on high-contention tables (`issues`, `ci_builds` — 8 tables total)
- **Soft deletes**: Some tables use `deleted_at` rather than hard deletes
- **Counter caches**: Denormalized counts (e.g., `merge_requests_count` on projects) for performance

## Traffic Distribution

| Category | DB Size | Write Traffic |
|---|---|---|
| CI | 35.7% | 49.0% |
| Merge Requests | 20.2% | 20.4% |
| Webhook logs | 22.4% | 2.8% |
| Everything else | 21.7% | 27.8% |

## Database Dictionary

GitLab maintains metadata about every table in `db/docs/` within the source repo. Each table entry documents:
- `table_name`, `feature_categories` (which feature owns it)
- `gitlab_schema` (which database it belongs to)
- `table_size` classification: small, medium, large, or over_limit
- `milestone` (version that introduced it)
- Sharding key information for multi-tenancy

## Key Sources

- Architecture overview: https://docs.gitlab.com/development/architecture/
- Database dictionary: https://docs.gitlab.com/development/database/database_dictionary/
- Schema file: https://gitlab.com/gitlab-org/gitlab/-/blob/master/db/structure.sql
- Database decomposition blog: https://about.gitlab.com/blog/2022/08/04/path-to-decomposing-gitlab-database-part1/
- Multiple databases docs: https://docs.gitlab.com/development/database/multiple_databases/
- Schema design analysis: https://shekhargulati.com/2022/07/08/my-notes-on-gitlabs-postgres-schema-design/
