# 💱 FX Pulse — Real-Time Currency Exchange Rate Pipeline

> An automated, serverless data pipeline that ingests live exchange rate data every hour, archives it to S3, and loads it into Snowflake for analytics — powered by AWS Lambda, EventBridge, and Snowflake Stored Procedures.

---

## 📐 Architecture Overview

```
OpenExchangeRates API
        │
        ▼
  AWS EventBridge (Hourly Cron)
        │
        ▼
  AWS Lambda (Python 3.11)
    ├── Fetches live rates from API
    ├── Reads credentials from AWS Secrets Manager
    ├──► S3 Bucket  →  exchange_rates/{year}/{month}/{day}/exchange-rates-{hour}.json
    └──► Snowflake  →  SP_EXCHANGE_RATE_LOADING(json_data, timestamp)
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
              RAW Table   STG Table  EXCHANGE_RATES
              (VARIANT)  (flattened)  (deduplicated)
```

---

## 🚀 Features

- **Fully Serverless** — No infrastructure to manage; runs entirely on AWS Lambda
- **Hourly Automation** — AWS EventBridge triggers the pipeline every hour
- **Dual Storage** — Raw JSON archived to S3 + structured data loaded into Snowflake
- **Idempotent Loads** — MERGE logic in Snowflake prevents duplicate records
- **Secure Credentials** — All secrets managed via AWS Secrets Manager (never hardcoded)
- **Partitioned S3 Storage** — Data organized by `year/month/day/hour` for efficient querying
- **Three-Layer Snowflake Architecture** — RAW → STG → EXCHANGE_RATES for clean data lineage

---

## ⚙️ AWS Infrastructure Setup

### 1. Lambda Function
- **Runtime:** Python 3.11
- **Architecture:** x86_64
- **Timeout:** Recommended 30–60 seconds
- **Memory:** 512 MB (recommended)

### 2. Lambda Layer (Dependencies)
Build the layer using Docker to ensure binary compatibility with Amazon Linux 2:

```bash
# Build using the exact Lambda base image
docker run --rm \
    -v $(pwd):/output \
    --platform linux/amd64 \
    --entrypoint pip \
    public.ecr.aws/lambda/python:3.11 \
    install requests snowflake-connector-python "pandas==2.2.3" "numpy==1.26.4" \
    -t /output/python

# Zip and upload
zip -r layer.zip python/
```

### 3. Environment Variables
Set the following in your Lambda function's configuration:

| Variable | Description | Example |
|---|---|---|
| `region_name` | AWS region | `us-east-1` |
| `snowflake_db` | Snowflake database name | `CURRENCY_DB` |
| `snowflake_role` | Snowflake role | `SYSADMIN` |
| `snowflake_wh` | Snowflake warehouse | `COMPUTE_WH` |
| `environment` | Deployment environment | `prod` |
| `s3_bucket_name` | Target S3 bucket | `fx-pulse-data` |
| `oer_base_url` | OpenExchangeRates API URL | `https://openexchangerates.org/api/latest.json` |
| `oer_app_id` | OpenExchangeRates App ID | `your_app_id` |
| `oer_base_currency` | Base currency | `USD` |

### 4. AWS Secrets Manager
Create a secret at `db/currency-echange-rate` with the following structure:

```json
{
  "fusion_snowflake": {
    "username": "your_snowflake_user",
    "password": "your_snowflake_password",
    "account_name": "your_account.region"
  }
}
```

### 5. IAM Permissions
Attach the following permissions to your Lambda execution role:

- AmazonEventBridgeFullAccess
- AmazonS3FullAccess
- AWSLambda_FullAccess
- SecretsManagerReadWrite

### 6. EventBridge Schedule
Create an EventBridge rule with a cron expression to trigger Lambda hourly:

```
rate(1 hour)
```

---

## ❄️ Snowflake Setup

Run the SQL in `snowflake/setup.sql` to create:

1. **`CURRENCY_DB`** — Database
2. **`CURRENCY`** — Schema
3. **`EXCHANGE_RATES_RAW`** — Stores raw API JSON response (VARIANT column)
4. **`EXCHANGE_RATES_STG`** — Flattened staging table with one row per currency pair
5. **`EXCHANGE_RATES`** — Final deduplicated table for analytics
6. **`SP_EXCHANGE_RATE_LOADING`** — Stored procedure that orchestrates the full load

### Stored Procedure Flow

```
SP_EXCHANGE_RATE_LOADING(p_json_data, p_datetime)
    │
    ├── TRUNCATE EXCHANGE_RATES_RAW
    ├── INSERT raw JSON into EXCHANGE_RATES_RAW
    ├── TRUNCATE EXCHANGE_RATES_STG
    ├── FLATTEN rates JSON → INSERT into EXCHANGE_RATES_STG
    └── MERGE EXCHANGE_RATES_STG → EXCHANGE_RATES (no duplicates)
```

---

## 📊 S3 Data Layout

Raw JSON files are stored with time-partitioned keys:

```
s3://your-bucket/
└── exchange_rates/
    └── 2025/
        └── 01/
            └── 15/
                ├── exchange-rates-00.json
                ├── exchange-rates-01.json
                ├── ...
                └── exchange-rates-23.json
```

---

## 🔍 Validation Queries

```sql
-- Check raw ingestion
SELECT * FROM CURRENCY.EXCHANGE_RATES_RAW;

-- Check flattened staging data
SELECT * FROM CURRENCY.EXCHANGE_RATES_STG;

-- Check final table
SELECT * FROM CURRENCY.EXCHANGE_RATES;

-- Verify hourly loads (should see one row group per hour)
SELECT timestamp_utc, COUNT(*) AS currency_pairs
FROM CURRENCY.EXCHANGE_RATES
GROUP BY timestamp_utc
ORDER BY timestamp_utc DESC;
```

---

## 🛠️ Tech Stack

| Component | Technology |
|---|---|
| Compute | AWS Lambda (Python 3.11) |
| Scheduler | AWS EventBridge |
| Secret Management | AWS Secrets Manager |
| Object Storage | AWS S3 |
| Data Warehouse | Snowflake |
| Data Source | OpenExchangeRates API |

---

## 📄 License

MIT License — feel free to use, modify, and distribute.
