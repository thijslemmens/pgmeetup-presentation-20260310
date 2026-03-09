---
marp: true
theme: contentgrid
paginate: false
---

<!-- _class: title -->

# Scaling Semantic Models

## Bespoke PostgreSQL Schemas at SaaS Scale

Thijs Lemmens · March 2026

---

# Agenda

1. The problem with generic data models
2. How ContentGrid generates schemas from semantic models
3. Automating DDL migrations at scale
4. Infrastructure patterns: isolation, performance, manageability
5. Lessons learned & trade-offs

---

<!-- _class: section -->

# The Problem

## Why generic EAV tables fall short

---

# The EAV Trap

## How most CMS platforms store content

<div class="columns">
<div>

### Generic EAV schema

```sql
CREATE TABLE alf_node (
  id      bigint PRIMARY KEY,
  type_qname_id bigint REFERENCES alf_qname
);
CREATE TABLE alf_qname (
  id        bigint PRIMARY KEY,
  local_name varchar(200)
);
CREATE TABLE alf_node_properties (
  node_id       bigint REFERENCES alf_node,
  qname_id      bigint REFERENCES alf_qname,
  string_value  varchar(1024),
  boolean_value bit,
  int_value     int8,
  double_value  float8,
  ...
);
```

</div>
<div>

### What you actually want

```sql
CREATE TABLE articles (
  id           uuid PRIMARY KEY,
  title        text        NOT NULL,
  published_at timestamptz,
  word_count   integer,
  author_id    uuid REFERENCES authors
);
```

</div>
</div>

---

# EAV: The Hidden Costs (1/2)

## Data integrity

| Concern               | EAV                                                  | Native schema               |
| --------------------- | ---------------------------------------------------- | --------------------------- |
| Type safety           | One column per type; wrong column = silent data loss | Enforced by the column type |
| Constraints           | `NOT NULL`, `UNIQUE` cannot be expressed             | Declarative, DB-enforced    |
| Referential integrity | Application-enforced                                 | Foreign keys                |

---

# EAV: The Hidden Costs (2/2)

## Performance & tooling

<style scoped>
section { font-size: 20px; }
</style>

| Concern               | EAV                                                                                                    | Native schema                  |
| --------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------ |
| Query complexity      | Reconstruct one node via dozens of `JOIN`s                                                             | Plain `SELECT`                 |
| Query planning        | Value-column statistics mix all property types — per-attribute selectivity is invisible to the planner | Accurate per-column statistics |
| Index efficiency      | Index on `(qname_id, string_value)` covers all attributes                                              | Targeted per-column indexes    |
| Tooling compatibility | Breaks ORMs, analytics tools                                                                           | Works out of the box           |
| Search performance     | Sync to Solr/Elasticsearch — operational heavy, denormalizes relations, eventual consistency   | Native full-text search, joins, transactional consistency |

> EAV trades correctness and performance for schema flexibility — but you can have both.

---

# How Bad Is It? A Real Migration

<div class="columns">
<div>

### The scenario

- **250 million** documents
- **60 attributes** each → 15 billion property rows
- EAV model on Oracle

</div>
<div>

### How large is the Oracle database?

| | |
|---|---|
| **A** | < 500 GB |
| **B** | 500 GB – 2 TB |
| **C** | 2 TB – 5 TB |
| **D** | > 5 TB |

</div>
</div>

---

<!-- _class: dark -->

# The EAV Oracle database: **6 TB**

- 15 billion property rows, each with 6+ typed value columns
- Indexes on `(qname_id, *_value)` for every attribute type
- Oracle block overhead, undo segments, redo logs multiply the footprint

> Now — same data, native PostgreSQL schema. Your guess?

---

# Same Data, Native PostgreSQL Schema

<div class="columns">
<div>

### What changes

- One row per document
- Typed columns — no attribute explosion
- No Oracle overhead

</div>
<div>

### How large is the PostgreSQL database?

| | |
|---|---|
| **A** | Still ~6 TB |
| **B** | 1 TB – 3 TB |
| **C** | 250 GB – 1 TB |
| **D** | < 250 GB |

</div>
</div>

---

<!-- _class: dark -->

# The ContentGrid PostgreSQL database: **~300 GB**

- 250M rows of typed data
- Targeted indexes
- No attribute row explosion, no Oracle overhead

<div class="highlight">

**20× smaller — by fixing the data model**

</div>

---

<!-- _class: section -->

# Our Approach

## Generating native schemas from semantic models

---

# Model-Driven Schema Generation

<style scoped>
pre { font-size: 15px; }
</style>

<div class="columns">
<div>

### User-defined semantic model

```json
{
  "name": "article",
  "table": "article",
  "attributes": [
    {
      "type": "simple",
      "name": "title",
      "dataType": "text",
      "columnName": "title",
      "constraints": [{ "type": "required" }]
    },
    {
      "type": "simple",
      "name": "publishedAt",
      "dataType": "datetime",
      "columnName": "published_at"
    },
    {
      "type": "simple",
      "name": "wordCount",
      "dataType": "long",
      "columnName": "word_count"
    }
  ],
  ...
}
```

</div>
<div>

### Generated PostgreSQL DDL

```sql
CREATE TABLE article (
  id           uuid PRIMARY KEY
                 DEFAULT gen_random_uuid(),
  title        text NOT NULL,
  published_at timestamptz,
  word_count   bigint
);
```

</div>
</div>

---

# One Schema per Tenant

## Full PostgreSQL schema isolation

<div class="columns">
<div>

- Each tenant gets its own PostgreSQL **schema** (`search_path`)
- Tables, indexes, and sequences are tenant-scoped
- No cross-tenant data leakage by construction
- Tenant schemas can be backed up, restored, or migrated independently

</div>
<div>

```
pg database
├── tenant_acme
│   ├── cg_article
│   ├── cg_author
│   └── cg_product
├── tenant_globex
│   ├── cg_post
│   └── cg_category
└── tenant_initech
    ├── cg_document
    └── cg_tag
```

</div>
</div>

---

<!-- _class: section -->

# Automating Migrations

## DDL at scale, safely

---

# The Migration Challenge

## Thousands of unique schemas, each evolving independently

- A model change → a DDL migration, applied **per tenant**
- Migrations must be **online** — no table locks on production traffic
- Schema drift: tenants may be on different model versions
- Failures must be recoverable without manual intervention

<div class="highlight">
At SaaS scale, "run migrations before deploy" is not an option. Migrations are a continuous background process.
</div>

---

# Migration Pipeline

```
Model change
    │
    ▼
DDL diff engine
(compare target model → current schema in pg_catalog)
    │
    ▼
Migration plan
(ordered list of safe DDL statements)
    │
    ▼
Execution queue
(per-tenant, rate-limited, retryable)
    │
    ├── ADD COLUMN / CREATE INDEX CONCURRENTLY / ...
    └── Track state in migration log table
```

---

# DDL Safety Rules

## What we allow — and what we don't

<div class="columns">
<div>

### Safe (online)

- `ADD COLUMN` with a default
- `CREATE INDEX CONCURRENTLY`
- `ALTER COLUMN` type widening (e.g. `varchar(50)` → `text`)
- `ADD FOREIGN KEY NOT VALID` + `VALIDATE CONSTRAINT`
- `DROP INDEX CONCURRENTLY`

</div>
<div>

### Requires care

- `DROP COLUMN` — deferred, soft-deleted first
- Column renames — additive alias approach
- `ALTER COLUMN SET NOT NULL` — backfill + constraint
- Any full table rewrite — avoided or done via shadow table

</div>
</div>

---

<!-- _class: section -->

# Infrastructure Patterns

## Keeping it performant and manageable

---

# Connection Isolation with PgBouncer

<div class="columns">
<div>

- Transaction-mode pooling per tenant schema
- `search_path` set at connection checkout
- Limits blast radius of a noisy tenant
- Dedicated pool config for migration workers vs. API workers

</div>
<div>

```ini
[pgbouncer]
pool_mode = transaction
default_pool_size = 10
max_client_conn = 1000

[tenant_acme]
dbname = contentgrid
auth_user = cg_tenant_acme
```

</div>
</div>

---

# Observability

## You can't manage what you can't measure

- **Migration lag** — how far behind is each tenant's schema?
- **Lock wait time** — are DDL statements blocking queries?
- **Schema size per tenant** — table bloat, index bloat
- **Query plans** — do generated indexes actually get used?

<div class="highlight-light">
We expose a <code>cg_migrations</code> table per tenant schema: every applied migration, its duration, and its status. Ops and tenants can both inspect it.
</div>

---

# Lessons Learned

- **Start with the catalog** — `pg_catalog` is your source of truth for schema drift, not your own migration log
- **Avoid locks** — every DDL statement goes through a lock-safety review; `CONCURRENTLY` is your friend
- **Rate-limit migrations** — running 10,000 `CREATE INDEX CONCURRENTLY` in parallel will hurt
- **Soft deletes before hard drops** — rename columns/tables with a tombstone prefix before dropping
- **Test with realistic tenant counts** — behaviour at 10 tenants and 10,000 tenants is very different

---

<!-- _class: dark -->

<style scoped>
section { font-size: 24px; }
</style>

# Trade-offs

## This approach is not free

- **Operational complexity** — migrations are a distributed system problem, not a one-shot script
- **Schema sprawl** — `pg_catalog` grows with every tenant; monitor `pg_class` row counts
- **Tooling gaps** — most migration frameworks assume one schema; we built our own diff engine
- **Upgrade coordination** — PostgreSQL major upgrades require validating all tenant schemas

> The payoff: every tenant gets a real relational model, real indexes, and real constraints — with none of the EAV query gymnastics.

---

<!-- _class: center -->

# Questions?

**contentgrid.com**

---

<!-- _class: title -->

# Thank You

## Thijs Lemmens

thijs.lemmens@xenit.eu
