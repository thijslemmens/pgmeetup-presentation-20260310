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

# The ContentGrid database: **~300 GB**

- 250M rows of typed data
- Targeted indexes
- No attribute row explosion, no Oracle overhead

<div class="highlight">

**20× smaller — by fixing the data model**

</div>

---

# Storage at a Glance

<style scoped>
.chart-wrap { display: flex; align-items: flex-end; justify-content: center; gap: 100px; height: 320px; margin-top: 60px; }
.bar-group { display: flex; flex-direction: column; align-items: center; justify-content: flex-end; gap: 8px; }
.bar-value { font-weight: 700; font-size: 22px; color: #084772; }
.bar { border-radius: 6px 6px 0 0; width: 160px; }
.bar-oracle { background: #ff0000; height: 200px; }
.bar-pg { background: #1db41d; height: 10px; }
.bar-label { font-size: 17px; font-weight: 600; color: #084772; text-align: center; line-height: 1.3; margin-top: 10px; }
.badge { background: #084772; color: #fff; border-radius: 20px; font-size: 14px; font-weight: 700; padding: 2px 12px; margin-top: 4px; display: inline-block; }
</style>

<div class="chart-wrap">
  <div class="bar-group">
    <div class="bar-value">6 TB</div>
    <div class="bar bar-oracle"></div>
    <div class="bar-label">Alfresco / Oracle<br><span class="badge">EAV</span></div>
  </div>
  <div class="bar-group">
    <div class="bar-value">~300 GB</div>
    <div class="bar bar-pg"></div>
    <div class="bar-label">ContentGrid / PostgreSQL<br><span class="badge">Native schema</span></div>
  </div>
</div>

---

<!-- _class: section -->

# Our Approach

## Generating native schemas from semantic models

---

# The Management Platform

<style scoped>
.flow { display: flex; align-items: stretch; gap: 0; margin: 28px 0 20px; }
.comp { background: #f5f8fc; border: 2px solid #d0e4f0; border-top: 4px solid #019ee3; border-radius: 8px; padding: 18px 16px; flex: 1; }
.comp h3 { color: #084772; margin: 0 0 6px 0; font-size: 0.95em; letter-spacing: 0.02em; }
.comp p { margin: 0; font-size: 0.72em; color: #5a7a95; line-height: 1.4; }
.arr { display: flex; align-items: center; padding: 0 10px; color: #019ee3; font-size: 1.6em; font-weight: 300; }
.tagline { background: #019ee3; color: #fff; border-radius: 8px; padding: 14px 24px; text-align: center; font-size: 0.88em; font-weight: 600; letter-spacing: 0.01em; }
</style>

<div class="flow">
  <div class="comp"><h3>Architect</h3><p>Source of truth for the domain model — entities, attributes, relations</p></div>
  <div class="arr">→</div>
  <div class="comp"><h3>Scribe</h3><p>Compiles model changes into:<br />
      &#x2022; JSON model<br />
      &#x2022; Flyway SQL migrations<br />
      &#x2022; OPA policies</p></div>
  <div class="arr">→</div>
  <div class="comp"><h3>Captain</h3><p>
  &#x2022; Provisions database <br />
  &#x2022; manages credentials <br />
  &#x2022; deployment</p></div>
</div>

<div class="tagline">Every model change → versioned, reviewed SQL migration &nbsp;·&nbsp; Schema always reflects the model — no manual DDL</div>

---

# The Runtime Platform

<style scoped>
.rt-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 32px; margin-top: 8px; }
.flow-col h3 { color: #084772; font-size: 0.9em; margin: 0 0 14px; border-bottom: 2px solid #d0e4f0; padding-bottom: 8px; }
.step { display: flex; align-items: flex-start; gap: 10px; margin-bottom: 10px; }
.step-icon { background: #019ee3; color: #fff; border-radius: 50%; width: 22px; height: 22px; font-size: 0.7em; font-weight: 700; display: flex; align-items: center; justify-content: center; flex-shrink: 0; margin-top: 2px; }
.step-text { font-size: 0.78em; color: #1a2b3c; line-height: 1.45; }
.step-text code { background: #f5f8fc; border: 1px solid #d0e4f0; border-radius: 4px; padding: 1px 5px; font-size: 0.9em; color: #084772; }
.callout { background: #084772; color: #fff; border-radius: 8px; padding: 11px 16px; font-size: 0.78em; font-weight: 600; margin-top: 14px; text-align: center; }
</style>

<div class="rt-grid">
<div class="flow-col">

### Dynamic SQL from the model

<div class="step"><div class="step-icon">1</div><div class="step-text">Model artifact loaded at startup — no fixed entity classes or ORM mappings</div></div>
<div class="step"><div class="step-icon">2</div><div class="step-text">JOOQ builds type-safe SQL at runtime from the active model</div></div>
<div class="step"><div class="step-icon">3</div><div class="step-text">Pagination, sorting, and relation traversal all generated dynamically</div></div>

</div>
<div class="flow-col">

### ABAC pushed down to SQL

<div class="step"><div class="step-icon">1</div><div class="step-text">OPA evaluates access policy via <strong>partial evaluation</strong> — without binding user data</div></div>
<div class="step"><div class="step-icon">2</div><div class="step-text">Returns a residual expression:<br><code>department = 'sales' OR status = 'published'</code></div></div>
<div class="step"><div class="step-icon">3</div><div class="step-text">App Server translates it into a SQL <code>WHERE</code> clause appended to every query</div></div>
<div class="callout">Unauthorized data never leaves the database</div>

</div>
</div>

---

# Partial Evaluation: The Key Insight

<style scoped>
.comparison { display: flex; gap: 2rem; margin: 1.2rem 0; }
.col { flex: 1; background: #f5f8fc; border-radius: 6px; padding: 1rem 1.2rem; }
.col.bad { border-top: 4px solid #c0392b; }
.col.good { border-top: 4px solid #019ee3; }
.col h3 { margin: 0 0 0.6rem; font-size: 0.9em; color: #1a2b3c; }
.col pre { font-size: 0.62em; margin: 0.4rem 0; background: #fff; border: 1px solid #d0e4f0; border-radius: 4px; padding: 0.5rem; color: #1a2b3c; }
.col .note { font-size: 0.68em; color: #1a2b3c; font-weight: 600; margin-top: 0.6rem; }
.highlight { background: #084772; color: #fff; border-radius: 6px; padding: 0.6rem 1rem; font-size: 0.78em; font-weight: 600; margin-top: 1rem; text-align: center; }
</style>

<div class="comparison">
<div class="col bad">

### Naïve approach

```
for each row in invoices:     ← full table scan
  if OPA.check(row, user):    ← N network calls
    return row
```

<div class="note">N rows fetched, N OPA calls, unauthorized data in memory</div>

</div>
<div class="col good">

### Partial evaluation

```
OPA.partial_eval(policy, user)  ← 1 call, no entity data
    → residual: dept='sales'
      OR status='published'

SELECT * FROM invoices
WHERE dept='sales'
  OR status='published'          ← filtered at the DB
```

<div class="note">1 OPA call, unauthorized rows never leave PostgreSQL</div>

</div>
</div>

<div class="highlight">The security boundary is enforced inside the database — not in application memory</div>

---

# From Policy Rule to SQL Filter

<style scoped>
.steps { display: flex; flex-direction: column; gap: 0.4rem; margin: 0.5rem 0; }
.row { display: grid; grid-template-columns: 13rem 1fr; align-items: center; background: #f5f8fc; border-left: 4px solid #019ee3; border-radius: 6px; padding: 0.5rem 1rem; gap: 1rem; }
.row-label { color: #084772; font-weight: 700; font-size: 0.8em; white-space: nowrap; }
.row pre { margin: 0; font-size: 0.72em; background: #fff; border: 1px solid #d0e4f0; border-radius: 4px; padding: 0.4rem 0.75rem; color: #1a2b3c; }
.note { font-size: 0.65em; color: #5a7a95; margin-top: 0.4rem; font-style: italic; }
.highlight-light { background: #084772; color: #fff; border-radius: 6px; padding: 0.55rem 1rem; font-size: 0.78em; font-weight: 600; margin-top: 0.6rem; text-align: center; }
</style>

<div class="steps">
<div class="row">
<div class="row-label">① Policy condition</div>
<pre>entity.department == user.department
OR entity.status == "published"</pre>
</div>
<div class="row">
<div class="row-label">② OPA partial eval</div>
<pre>OPA.partial_eval(policy, user)
  → dept == user.dept  OR  status == "published"</pre>
</div>
<div class="row">
<div class="row-label">③ SQL WHERE (JOOQ)</div>
<pre>SELECT * FROM invoices
WHERE department = 'sales' OR status = 'published'</pre>
</div>
</div>

<div class="note">Defined in Architect UI → compiled to Rego by Scribe → evaluated by OPA (1 call, result in Gateway JWT) → App Server builds JOOQ predicate</div>
<div class="highlight-light">Unauthorized rows never leave the database — the filter runs inside PostgreSQL</div>

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

<!-- _class: "" -->
<style scoped>
.layout { display: flex; gap: 2em; align-items: flex-start; margin-top: 0.4em; }
.layout > .left { flex: 0 0 42%; }
.layout > .right { flex: 1; }
.chain-box {
  background: #f5f8fc; border-radius: 6px;
  padding: 0.55em 0.9em 0.5em; margin-bottom: 0;
}
.chain-box.api  { border-top: 4px solid #019ee3; }
.chain-box.view { border-top: 4px solid #084772; }
.chain-box.base { border-top: 4px solid #0e7c6b; }
.chain-box .title { font-weight: 700; font-size: 0.82em; color: #0f172a; }
.chain-box .sub   { font-size: 0.68em; color: #475569; margin-top: 0.15em; }
.arr { text-align: center; font-size: 1.1em; color: #94a3b8;
  line-height: 1.4; margin: 0.15em 0; }
.col-label { font-size: 0.7em; font-weight: 600; color: #1e3a5f; margin-bottom: 0.35em; }
.code-block { background: #f8fafc; color: #1a1a2e; border-left: 4px solid #2563eb;
  border-radius: 4px; padding: 0.65em 0.9em; font-family: monospace;
  font-size: 0.58em; line-height: 1.6; }
.hl { background: #fef9c3; border-radius: 2px; }
.callout { background: #0f172a; color: #f1f5f9; border-radius: 5px;
  padding: 0.45em 1.2em; margin-top: 0.75em; font-size: 0.68em;
  font-style: italic; text-align: center; }
</style>

# API Schema Versioning

## Each released API version gets its own PostgreSQL schema

<div class="layout">
<div class="left">

<div class="chain-box api">
  <div class="title">API client</div>
  <div class="sub">search_path = "V3"</div>
</div>
<div class="arr">▼</div>
<div class="chain-box view">
  <div class="title">Schema "V3"</div>
  <div class="sub">compatibility view<br>projects old column set</div>
</div>
<div class="arr">▼</div>
<div class="chain-box base">
  <div class="title">Base tables</div>
  <div class="sub">current physical schema</div>
</div>

</div>
<div class="right">

<div class="col-label">Dropped column → null cast</div>
<pre class="code-block">CREATE OR REPLACE VIEW "V3"."claim_document" AS
  SELECT id, _version,
    claim_number, source,
    <span class="hl">null::timestamptz  -- "date_received" was dropped</span>
      AS date_received,
    audit__created_by_id
  FROM claim_document;  -- base table</pre>

</div>
</div>

<div class="callout">INSTEAD OF triggers make views fully writable — no app code changes needed.</div>

---

<!-- _class: "" -->
<style scoped>
.cards { display: grid; grid-template-columns: repeat(3, 1fr);
  gap: 1.1em; margin-top: 0.5em; }
.card { background: #f5f8fc; border-radius: 7px; padding: 0.7em 0.85em 0.75em; }
.card.blue   { border-top: 4px solid #019ee3; }
.card.teal   { border-top: 4px solid #0e7c6b; }
.card.amber  { border-top: 4px solid #d97706; }
.card-icon { font-size: 1.3em; line-height: 1; margin-bottom: 0.2em; }
.card-title { font-weight: 700; font-size: 0.78em; color: #0f172a;
  margin-bottom: 0.35em; }
.code-block { background: #0f172a; color: #e2e8f0; border-radius: 4px;
  padding: 0.5em 0.75em; font-family: monospace;
  font-size: 0.58em; line-height: 1.55; margin: 0.4em 0; }
.pills { display: flex; flex-wrap: wrap; gap: 0.3em; margin: 0.4em 0; }
.pill { background: #e0f2fe; color: #0369a1; border-radius: 99px;
  padding: 0.1em 0.55em; font-size: 0.58em; font-weight: 600; font-family: monospace; }
.step-row { display: flex; align-items: baseline; gap: 0.45em;
  font-size: 0.64em; margin: 0.3em 0; color: #1e293b; }
.step-badge { background: #084772; color: #fff; border-radius: 50%;
  width: 1.4em; height: 1.4em; display: flex; align-items: center;
  justify-content: center; font-size: 0.78em; font-weight: 700;
  flex-shrink: 0; }
.card-tag { font-size: 0.6em; color: #64748b; font-style: italic;
  margin-top: 0.5em; }
.callout { background: #0f172a; color: #f1f5f9; border-radius: 5px;
  padding: 0.45em 1.2em; margin-top: 0.75em; font-size: 0.68em;
  font-style: italic; text-align: center; }
</style>

# Safe Schema Evolution Patterns

## Generated migrations follow PostgreSQL best practices

<div class="cards">

<div class="card blue">
  <div class="card-icon">🚫🔒</div>
  <div class="card-title">CREATE INDEX CONCURRENTLY</div>
  <div style="font-size:0.63em;color:#334155">No table lock</div>
  <pre class="code-block">CREATE INDEX CONCURRENTLY idx
  ON t (normalize(col, NFKC));</pre>
  <div class="card-tag">Separate migration file</div>
</div>

<div class="card teal">
  <div class="card-icon" style="font-size:1.5em;font-family:serif">ƒ(x)</div>
  <div class="card-title">IMMUTABLE index function</div>
  <div class="pills">
    <span class="pill">IMMUTABLE</span>
    <span class="pill">PARALLEL SAFE</span>
    <span class="pill">RETURNS NULL ON NULL INPUT</span>
  </div>
  <pre class="code-block">lower(normalize(unaccent(arg), NFKC))</pre>
  <div class="card-tag">Safe in index expressions</div>
</div>

<div class="card amber">
  <div class="card-icon">🛡️</div>
  <div class="card-title">Safe NOT NULL</div>
  <div class="step-row"><div class="step-badge">1</div><code style="font-size:0.95em">ADD COLUMN col text NULL</code></div>
  <div class="step-row"><div class="step-badge">2</div><code style="font-size:0.95em">UPDATE … SET col = …</code></div>
  <div class="step-row"><div class="step-badge">3</div><code style="font-size:0.95em">SET NOT NULL</code></div>
  <div class="card-tag">Zero-downtime constraint</div>
</div>

</div>

<div class="callout">All patterns generated automatically — the model describes intent, the platform generates safe SQL.</div>

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
