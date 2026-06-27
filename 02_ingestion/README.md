# Stage 2: Ingestion

## Goal
Load raw eCommerce event data from Kaggle into Databricks for further processing.
Data must be stored in a structured, queryable format -  not just as files.

## Architecture Decision
Ingestion follows a two-step pattern recommended by Databricks:

Kaggle (source) → Volume (landing zone) → Delta Tables (structured storage)

**Why two steps?**
Volumes are designed for raw file storage - CSV, JSON, Parquet - before transformation begins.
Delta tables provide structure, schema enforcement, ACID transactions, and fast SQL queries.
Mixing these responsibilities would violate separation of concerns in a data pipeline.

## Step 1: Kaggle API → Volume (Landing Zone)

**What:** Download raw CSV files programmatically and store in Databricks Volume.

**Why Volume?** It serves as a landing zone - a temporary holding area for raw files
before they are processed. This is a standard pattern in lakehouse architecture.

**Challenge - Token Security:**
Kaggle API requires authentication. Hardcoding the token in notebook code is a security risk
and cannot be committed to GitHub. Several approaches were tested:

| Approach | Result | Reason |
|----------|--------|--------|
| Databricks Secrets (CLI) | Failed | Feature not available in Community Edition |
| dotenv from local .env file | Failed | Databricks runs in cloud - no access to local machine |
| Databricks Widget | ✅ Success | Token entered at runtime, never stored in code |

**Final approach:** Token is passed via Databricks Widget - a UI input field that appears
at the top of the notebook. Value is read into os.environ at runtime and never persisted.

**Challenge - File Transfer:**
Kaggle CLI cannot write directly to Databricks Volume. Several approaches were tested:

| Approach | Result | Reason |
|----------|--------|--------|
| os.makedirs to /dbfs/ | Failed | DBFS root disabled in Community Edition |
| dbutils.fs.cp from /tmp/ | Failed | Local filesystem access denied |
| Download to /tmp/ + shutil.copy to Volume | ✅ Success | Python native file copy works |

**Result:**
| File | Size | Location |
|------|------|----------|
| 2019-Oct.csv | 5.6 GB | /Volumes/workspace/default/ecommerce_raw/ |
| 2019-Nov.csv | 9.0 GB | /Volumes/workspace/default/ecommerce_raw/ |

## Step 2: Volume → Delta Tables (Structured Storage)

**What:** Read CSV files from Volume and create permanent Delta tables in Unity Catalog.

**Why Delta tables?**
Delta Lake format provides ACID transactions, schema enforcement, time travel,
and fast SQL queries - essential for reliable data pipelines.

**Unity Catalog Structure created:**
Catalog: ecommerce_pipeline

├── Schema: raw          ← raw data as ingested, no transformations

├── Schema: staging      ← cleaned and typed data (Stage 4)

├── Schema: intermediate ← business logic applied (Stage 5)

└── Schema: mart         ← final tables for reporting (Stage 5)

All schemas created upfront to establish clear pipeline architecture from the start.

**Method:** CREATE TABLE AS SELECT using read_files() function.

Note: Standard CTAS with OPTIONS syntax failed with a parse error in this Databricks version.
read_files() is the correct approach for reading external files in Unity Catalog context.

**Result:**
| Table | Rows | Schema |
|-------|------|--------|
| ecommerce_pipeline.raw.events_oct_2019 | 42,448,764 | ecommerce_pipeline.raw |
| ecommerce_pipeline.raw.events_nov_2019 | 67,501,979 | ecommerce_pipeline.raw |
| **Total** | **109,950,743** | |

**Schema inferred:**
| Column | Type | Notes |
|--------|------|-------|
| event_time | timestamp | Parsed correctly from UTC string |
| event_type | string | Values: view, cart, purchase (remove_from_cart to verify) |
| product_id | int | 7-8 digit values observed - verify range in Stage 3 |
| category_id | bigint | 19-digit values - bigint correct |
| category_code | string | Nullable - expected per documentation |
| brand | string | Nullable - expected per documentation |
| price | double | Always present per documentation |
| user_id | int | 9-digit values observed - verify if int is sufficient in Stage 3 |
| user_session | string | GUID format, resets after long pause |
| _rescued_data | string | 0 rows - all data parsed without issues |

## Open Questions Answered
- **Q:** What does Kaggle API link return?
  **A:** Same 2 files - 2019-Oct.csv and 2019-Nov.csv. No additional data via API.

## Confirmed From Stage 1
- category_code contains NULL values - confirmed in raw table
- brand contains NULL values - confirmed in raw table
- category_code has varying hierarchy levels - confirmed (2 and 3 levels observed)

## Open Questions for Stage 3
1. How many category_code values have 2 levels vs 3 levels?
2. Are product_id and user_id safely stored as int, or do they need bigint?
3. Does remove_from_cart appear in event_type column in full dataset?
4. Are there additional months available beyond Oct-Nov 2019?