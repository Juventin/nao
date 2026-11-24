# Context

## RULES

- Always answer in the same language as the user question.
- If the question is trivial and needs a short answer, be concise and just give the KPI asked. 
- Custom skills in agent/skills/ should be accessed via the read tool if not auto-loaded via a / trigger, rather than using load_skill.
- Before querying anything, identify which **Project Scope** (below) the question belongs to, and stay confined to it.

## Project Scopes — decide this before querying anything

The data platform serves three separate data products. They have different audiences, different tables. Pick exactly one scope per question, based on what the question is actually about — not on what data could theoretically answer it.

| Scope | When | Tables | Entry point |
|---|---|---|---|
| **General BI** (default) | Anything general: Looker, stores, GMV, modules, transactions... | `ps-data-dataform` (all `*_prod` except `hubspot_prod.scoring_agencies`), `ps-data-science-dataform`, `ps-data-ascend-prod`, `ps-data-prestashop-airbyte`, Looker | rest of this file, `agent/skills/looker-store-observatory*.md` |
| **Partner Dashboard** | Partner Portal, Agency Cockpit, partner module perf/competitive position/ecosystem stats | `ps-data-agency-cockpit.partners_prod`, `.ps_ecosystem_prod` **only** | `agent/skills/partner-data-dashboard.md` |
| **Expert Scoring** | Agency/expert scoring, points, badges, per-agency KPIs | `ps-data-dataform.hubspot_prod.scoring_agencies` + supporting tables in the skill | `agent/skills/scoring-agencies.md` |

Rules:
- No clear Partner Dashboard / Expert Scoring mention → General BI. Don't open those skills "just in case".
- Once scoped, stay in that scope's tables/skill.
- Partner Dashboard & Expert Scoring can pull from `ps-data-dataform`. Not the reverse.

## Data platform architecture

Looker (`ps-data-looker`) = main entry point, built mostly on `ps-data-dataform`.

- **ps-data-dataform** — main project, feeds Looker. Built by Dataform repo `ps-data-dataform`.
- **ps-data-science-dataform** — ML/data science project, own repo, own BQ project. Feeds `ps-data-dataform` and Looker.
- **ps-data-prestashop-airbyte** — raw Airbyte ingestion, no repo, upstream of `ps-data-dataform`.
- **ps-data-ascend-prod** — legacy pre-Dataform transforms/referentials, still used in Looker/`ps-data-dataform`, no longer refreshed.
- **ps-data-hightouch** — activation project (repo `ps-data-hightouch-dataform`). Feeds Hightouch → HubSpot/Zendesk. Not in Looker. Inputs: `ps-data-dataform`, Airbyte raw (HubSpot/Zendesk/Chargebee/expert portal), Datastream, legacy Ascend/Stripe.
- **ps-data-agency-cockpit** — Partner Dashboard project (datasets `partners`, `ps_ecosystem`, repo `ps-data-agency-cockpit-dataform`), reads from `ps-data-dataform`. Feeds Superset/Agency Cockpit. Not in Looker.

### Data flow

```
Main:
ps-data-prestashop-airbyte → ps-data-dataform → ps-data-looker

Other flows : 

ps-data-science-dataform    (ML/science transformations — repo: ps-data-science-dataform)
    └──→ ps-data-dataform

ps-data-ascend-prod   (Legacy data transformations and referentials)
    └──→ ps-data-dataform
    └──→ ps-data-looker

ps-data-hightouch   (CRM activation transformations — repo: ps-data-hightouch-dataform)
    └──→ Hightouch (sync) ──→ HubSpot / Zendesk

ps-data-agency-cockpit   (Partner Dashboard transformations — repo: ps-data-agency-cockpit-dataform)
    └──→ Superset (embedded in the ps-data-agency-cockpit service) ──→ Partner Dashboard
```

### Repo → BigQuery project

| Repo | Role | BigQuery project |
|---|---|---|
| `ps-data-dataform` | SQL transformations (Dataform) | `ps-data-dataform` |
| `ps-data-science-dataform` | ML/science transformations (Dataform) | `ps-data-science-dataform` |
| `ps-data-hightouch-dataform` | CRM activation transformations feeding Hightouch → HubSpot/Zendesk (Dataform) | `ps-data-hightouch` |
| `ps-data-agency-cockpit-dataform` | Partner Dashboard transformations feeding Superset (Dataform) | `ps-data-agency-cockpit` |
| `ps-data-looker` | LookML data model and explores | reads from `ps-data-dataform` and `ps-data-science-dataform` |
| *(no repo)* | Raw ingestion via Airbyte | `ps-data-prestashop-airbyte` |
| *(no repo)* | Legacy data transformations and referentials (before Dataform) | `ps-data-ascend-prod` |

## Tooling

### Execute SQL

- When using `execute_sql`, the `save_to` parameter only works when querying the `duckdb_local` database.
- To keep a warehouse result (BigQuery), first run the query without `save_to`, then re-run it against `duckdb_local` as `SELECT * FROM <query_id>` using the `save_to` parameter.

### Looker MCP

Use the Looker MCP **only** when the question is specifically about Looker, or contains a Looker link (`https://prestashop.cloud.looker.com/...`). In any other case — generic data questions, BigQuery/Dataform questions, exploration of tables, etc. — answer from BigQuery and the Dataform repo context instead.

Default folder for looks/dashboards: `folder_id = 1017`.

## Business vocabulary — aliases to resolve before querying

- **"M1", "Micro", "Small"** = **store size** segments (based on trailing-12-months CloudSync GMV). Any question that filters or segments on these terms (e.g. "how many M1 on PS 9.2?") must read `semantics/store-size.md` and filter on `store_size_l12m` — never treat "M1" as a version, a module or a plain count of active stores.
- **"PS 9", "PS 9.2", "version 9.2"** = PrestaShop version of a store. See `semantics/store-ps-version.md`.

## Topic-specific rules: `semantics/` and `queries/`

Check both folders for every request before answering — don't assume uncovered without checking.

- `semantics/*.md` — business definitions, authoritative table/column, pitfalls, sanity magnitudes.
- `queries/*.sql` — ready-to-adapt reference queries.

### Query performance & partitioning

BigQuery queries are subject to a processing limit of 3 GB per call. To avoid errors:
- **Always filter on partition columns**: Most large tables (e.g., `cloudsync_orders`, `ps_checkout_orders`, `store_activity_v2`, `pspay_gold_prod.payment_intent`) are partitioned by date/month (often `created_at` or `month_year`). Always include a filter on these columns.
- **Prune partitions early**: Use `TIMESTAMP(date)` or `DATETIME(date)` to ensure filters match the partition column type exactly.
- **Avoid SELECT ***: Only select the columns you need for the analysis.
- **Use Sampling**: For exploratory analysis on huge tables, use `TABLESAMPLE SYSTEM (1 PERCENT)` or a small `LIMIT`.
