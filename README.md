# ipl-analytics-lakehouse
# IPL Cricket Analytics Lakehouse

End-to-end medallion lakehouse built on Databricks + Delta Lake, analyzing 
17 seasons of IPL cricket data (2008–2024). Includes a natural language 
query layer powered by LangChain for plain-English data exploration.

**Status:** Bronze and Silver layers complete. Gold layer and GenAI query 
engine in progress.

## Architecture

Kaggle CSVs → Bronze (raw Delta) → Silver (cleaned, validated) → 
Gold (business aggregates) → GenAI Query Layer

[Insert architecture diagram here — draw.io export]

## Tech Stack

Databricks Community Edition · Delta Lake · PySpark · Unity Catalog · 
LangChain · OpenAI

## What's built so far

### Bronze layer
Raw ingestion of `matches.csv` (1,095 rows) and `deliveries.csv` (260,920 rows) 
into Delta tables, with full traceability via three metadata columns added 
to every row:

- `ingested_at` — timestamp of ingestion
- `source_file` — originating CSV file
- `ingestion_id` — unique ID per pipeline run, shared across all tables 
  written in that run

Every run also logs row counts and status to a `pipeline_audit` Delta 
table, giving a single queryable view across the whole pipeline without 
inspecting each table's Delta transaction log individually.

### Silver layer
Cleaned, validated, and joined data:

- Standardised legacy team names (e.g. "Delhi Daredevils" → 
  "Delhi Capitals") using a Spark map-based lookup
- Fixed malformed string values ("NA") in numeric columns using 
  `try_cast()` instead of `cast()`, converting unparseable values to 
  NULL rather than failing the pipeline
- Derived analytical flags on every delivery: `is_boundary`, `is_six`, 
  `is_dot_ball`, `is_death_over`, `is_powerplay`, `is_legal_delivery`
- Joined 260,920 delivery records against match data — validated zero 
  orphan records before writing
- `silver.deliveries` partitioned by `season_year` for query performance

## Data quality approach

Every layer validates before writing:

- Bronze logs ingestion counts to an audit table on every run
- Silver checks referential integrity explicitly — counts orphan 
  records after every join rather than assuming the join succeeded
- Null handling is deliberate, not silent — nulls are replaced with 
  meaningful values ("No Result", "N/A") only where business logic 
  supports it, never blindly dropped

## Key technical decisions

**Why Delta Lake over plain Parquet** — ACID transactions, time travel 
via `DESCRIBE HISTORY`, and schema enforcement matter for a pipeline 
that gets re-run repeatedly during development.

**Why try_cast() over cast()** — source data quality is never guaranteed. 
A pipeline that crashes on the first malformed value isn't production-ready.

**Why partition deliveries by season_year** — 260K rows across 17 seasons; 
most queries filter by season, so this directly reduces scan volume.

## What's next

- Gold layer: player stats, team analytics, season summaries
- Natural language query engine using LangChain + Delta tables
- Architecture diagram and full data dictionary

## How to run

1. Clone this repo
2. Create a Databricks Community Edition account
3. Download the IPL dataset from Kaggle, upload to a Unity Catalog volume
4. Run notebooks in order: `01_bronze` → `02_silver`
5. (Gold and GenAI layers — coming soon)
