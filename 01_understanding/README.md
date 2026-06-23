# Stage 1: Understanding the Data

## Dataset

**Name:** eCommerce behavior data from multi-category store

**Source:** Kaggle — mkechinov/ecommerce-behavior-data-from-multi-category-store

**Provider:** REES46 Marketing Platform (rees46.com)

## Available Data
Kaggle hosts 2 files (max 20GB limit per dataset):
- 2019-Oct.csv
- 2019-Nov.csv

Additional months (Dec 2019 — Apr 2020) available via direct links in dataset description.
Decision on scope (how many months to include) — to be made in Stage 2.

## Schema
| Column | Type | Description |
|--------|------|-------------|
| event_time | timestamp | Time of event in UTC |
| event_type | string | view, cart, remove_from_cart, purchase |
| product_id | int | Product identifier (7-8 digits observed) |
| category_id | int | Category identifier (19 digits observed) |
| category_code | string | Category hierarchy l1.l2.l3 — can be missing for accessories (expected behavior per documentation) |
| brand | string | Lowercase brand name — can be missing |
| price | float | Product price — always present |
| user_id | int | Permanent user identifier (9 digits observed) |
| user_session | string | Temporary session ID (GUID) — resets after long pause |

## Event Types
- view — user viewed a product
- cart — user added product to cart
- remove_from_cart — user removed product from cart
- purchase — user purchased a product

Note: first inspection of raw files showed only view events in first 5 rows.
remove_from_cart presence in full dataset — to be verified in Stage 3.

## Row Count (local files)
| File | Rows |
|------|------|
| 2019-Oct.csv | 42,448,764 |
| 2019-Nov.csv | 67,501,979 |
| **Total** | **109,950,743** |

## Row Count Discrepancy
- Kaggle dataset description: 85M rows
- Row Zero (via API link): 67,501,979 rows
- Local files total: 109,950,743 rows

Kaggle API link used in Row Zero:
https://www.kaggle.com/api/v1/datasets/download/mkechinov/ecommerce-behavior-data-from-multi-category-store

Direct file links found in dataset description:
https://data.rees46.com/datasets/marketplace/2019-Oct.csv.gz
https://data.rees46.com/datasets/marketplace/2019-Nov.csv.gz

Hypothesis: API link may return a different or partial dataset.
To be verified in Stage 2 — compare API download vs direct file download.
Any discrepancy must be explicitly documented in the analysis.

## Key Observations
- Dataset is an event-level behavioral log, not a product catalog
- Base unit is a single user interaction with a product at a point in time
- category_code has up to 3 hierarchy levels separated by a dot (e.g. appliances.kitchen.washer)
- Missing category_code is expected behavior for accessories per official documentation
- Missing brand is documented as possible
- One session can contain multiple purchase events — this is a single order, not duplicates
- user_session resets when user returns after a long pause

## Open Questions
1. What does the Kaggle API link actually return — same files or different dataset?
2. How many category_code values have 2 levels vs 3 levels?
3. How many distinct values of product_id length exist (7 digits, 8 digits, other)?
4. Does 'remove_from_cart' value appear in event_type column in the downloaded files?
5. How to handle event_time split — detailed columns (year, month, week, day, hour) or simple date + time — decision depends on semantic layer design
6. Primary/foreign key strategy — depends on whether star schema is chosen

## Links
- Dataset page: https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store
- Oct file: https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store?select=2019-Oct.csv
- Nov file: https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store?select=2019-Nov.csv
- API download: https://www.kaggle.com/api/v1/datasets/download/mkechinov/ecommerce-behavior-data-from-multi-category-store
- Direct Oct: https://data.rees46.com/datasets/marketplace/2019-Oct.csv.gz
- Direct Nov: https://data.rees46.com/datasets/marketplace/2019-Nov.csv.gz