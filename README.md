# Airbnb Analytics with dbt & Snowflake

This repository contains an end-to-end analytics engineering project built with dbt Core on Snowflake, based on the public Airbnb dataset used in the “Complete dbt Bootcamp (Zero to Hero)” course.

The goal of this project is to model, test, and document Airbnb listings, hosts, and reviews data and to produce analytics-ready tables for downstream dashboards and analysis.

## 1. Tech Stack

* Warehouse: Snowflake
* Transformation: dbt Core (v1.10) + dbt-snowflake
* Language: SQL + Jinja
* Orchestration (optional): Dagster / dbt CLI
* Version control: Git + GitHub

## 2. Data Pipeline Overview

Raw CSV files are loaded into Snowflake in the AIRBNB.RAW schema:

* RAW_LISTINGS
* RAW_HOSTS
* RAW_REVIEWS

Using dbt, these are transformed step-by-step into:

1. Staging / Source models (models/src/)
2. Cleansed dimension models (models/dim/)
3. Fact model for reviews (models/fct/)
4. Mart model for business analysis (models/mart/)
5. Snapshots to track slowly changing dimensions (snapshots/)

The final tables are exposed to BI tools (e.g. Preset / Superset) for dashboards.

## 3. Project Structure

High-level layout of the project:

* models/

  * src/: source models over raw Snowflake tables
  * dim/: cleaned dimensions for listings and hosts
  * fct/: fact model for reviews
  * mart/: mart layer for business analysis (full moon reviews)
  * schema.yml, sources.yml, docs.md, overview.md
* seeds/: seed_full_moon_dates.csv
* snapshots/: SCD snapshots for hosts and listings
* tests/: generic and singular tests
* analyses/: ad-hoc analysis queries
* dbt_project.yml: main dbt config file

## 4. Data Modeling

### 4.1 Source Models (models/src/)

These models provide a thin layer over the raw Snowflake tables and rename columns:

* src_listings

  * Selects from AIRBNB.RAW.RAW_LISTINGS
  * Renames id → listing_id, name → listing_name, price → price_str, etc.

* src_hosts

  * Selects from AIRBNB.RAW.RAW_HOSTS
  * Renames id → host_id, name → host_name

* src_reviews

  * Selects from AIRBNB.RAW.RAW_REVIEWS
  * Renames date → review_date, comments → review_text, etc.

### 4.2 Dimension Models (models/dim/)

* dim_listings_cleansed

  * Cleans minimum_nights (replaces 0 with 1)
  * Casts price_str to a numeric price
  * Keeps key attributes: room_type, host_id, timestamps

* dim_hosts_cleansed

  * Cleans host_name (NULL → "Anonymous")
  * Ensures one row per host_id
  * Uses a model contract enforced via schema.yml (types defined for each column)

* dim_listings_w_hosts

  * Joins listings and hosts
  * Adds host_is_superhost flag
  * Computes a combined updated_at using the greatest of listing and host timestamps

### 4.3 Fact Model (models/fct/fct_reviews.sql)

fct_reviews is an incremental model:

* Selects from src_reviews
* Filters out reviews with empty review_text
* Optionally generates a surrogate review_id using dbt_utils.generate_surrogate_key
* Uses is_incremental() logic to only load new rows based on review_date

This demonstrates how to handle large, append-only tables efficiently in dbt.

### 4.4 Mart Model (models/mart/mart_fullmoon_reviews.sql)

Combines:

* fct_reviews
* seed_full_moon_dates (from seeds/)

It flags whether each review happened on the night after a full moon, exposing an is_full_moon column ("full moon" / "not full moon").
This model is used for downstream analysis and unit testing.

## 5. Testing Strategy

The project uses multiple layers of testing:

### 5.1 Generic & Column Tests (models/schema.yml)

Examples:

* Uniqueness & not null

  * dim_listings_cleansed.listing_id → unique, not_null
  * dim_hosts_cleansed.host_id → unique, not_null

* Relationships

  * dim_listings_cleansed.host_id must exist in dim_hosts_cleansed.host_id
  * fct_reviews.listing_id must exist in dim_listings_cleansed.listing_id

* Accepted values

  * room_type must be one of:

    * Entire home/apt, Private room, Shared room, Hotel room
  * is_superhost must be either t or f
  * review_sentiment must be one of: positive, neutral, negative

* Custom generic test: positive_values

  * Defined in tests/generic/positive_values.sql
  * Ensures that minimum_nights is always greater than 0

### 5.2 Storing Test Failures

In dbt_project.yml, data tests are configured to store failing rows into a dedicated schema for easier investigation:

* store_failures: true
* schema: _test_failures

### 5.3 Singular Tests

* tests/consistent_created_at.sql

  * Ensures no review is created before its listing was created
  * Joins dim_listings_cleansed and fct_reviews on listing_id and checks that created_at is always earlier than review_date

### 5.4 Unit Test (dbt v1.10+)

* Defined in models/mart/unit_tests.yml
* Provides mock rows for fct_reviews and seed_full_moon_dates
* Verifies that the is_full_moon flag behaves as expected around a specific full moon date

## 6. Snapshots

Snapshots are defined under snapshots/:

* scd_raw_listings (based on source('airbnb', 'listings'))
* scd_raw_hosts (based on source('airbnb', 'hosts'))

They use:

* Strategy: timestamp
* Unique key: id
* Change column: updated_at
* hard_deletes: invalidate

This allows tracking historical changes to listings and hosts over time.

## 7. Documentation

dbt’s documentation features are used via:

* docs.md – description blocks such as dim_listing_cleansed__minimum_nights
* overview.md – high-level project overview and input schema image
* Column-level descriptions in schema.yml

Docs can be generated and viewed with:

* dbt docs generate
* dbt docs serve

## 8. How to Run This Project

Prerequisites:

* Python 3.12
* dbt-core and dbt-snowflake installed
* A Snowflake account with an AIRBNB database and RAW / DEV schemas
* A profiles.yml configured with the Snowflake connection and a DEV target

Basic commands:

* Install dependencies: pip install dbt-core dbt-snowflake
* Run all models: dbt run
* Run tests: dbt test
* Generate docs: dbt docs generate

## 9. Possible Extensions

Ideas to extend this project:

* Add business metrics such as revenue per night or occupancy rate
* Create additional marts for host performance or review quality
* Build BI dashboards on top of dim_listings_w_hosts, fct_reviews, and mart_fullmoon_reviews
* Orchestrate dbt runs using Dagster, Airflow, or dbt Cloud

## 10. Purpose

This project is designed as a hands-on analytics engineering portfolio piece to:

* Practice dbt and Snowflake on a realistic dataset
* Apply data modeling, testing, and documentation best practices
* Demonstrate familiarity with incremental models, snapshots, contracts, tests, and dbt docs


## Project Screenshots

### dbt Docs – Data Lineage


![dbt](dbt%20lineage.png)


### Snowflake Tables


![snowflake](snowflake%20databases.png)


### Project Structure


![vscode project](vscode%20airbnb.png)
