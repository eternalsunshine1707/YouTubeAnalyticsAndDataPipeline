# YouTube Analytics & Data Pipeline

An end-to-end data engineering project that ingests, transforms, and analyzes YouTube trending video data across multiple regions using AWS-native services. Raw CSV and JSON data are processed through a multi-layer pipeline and visualized in a QuickSight dashboard.

---

## Architecture Overview

```
Kaggle Dataset (CSV + JSON)
        |
        v
   AWS S3 (Raw Layer)
   - region-partitioned CSVs
   - JSON reference data
        |
        v
  AWS Glue + Lambda
  - JSON → Parquet (Lambda)
  - CSV → Parquet (Glue ETL)
  - Schema cataloging (Glue Crawler)
        |
        v
   AWS S3 (Cleaned Layer)
        |
        v
  AWS Athena (SQL querying)
        |
        v
  Amazon QuickSight (Dashboard)
```

---

## Tech Stack

| Service | Role |
|---|---|
| AWS S3 | Raw and cleaned data storage with Hive-style partitioning |
| AWS Lambda | Event-driven JSON to Parquet conversion |
| AWS Glue | ETL jobs for CSV to Parquet, Glue Crawler for schema cataloging |
| AWS Athena | Serverless SQL querying on the cleaned layer |
| Amazon QuickSight | Business intelligence dashboard |
| AWS IAM | Role-based access control across services |
| Python (PySpark, pandas, awswrangler) | Transformation logic |

---

## Dataset

Source: [Kaggle - Trending YouTube Video Statistics](https://www.kaggle.com/datasets/datasnaek/youtube-new)

The dataset contains trending video statistics across 10 regions: CA, DE, FR, GB, IN, JP, KR, MX, RU, and US. It includes fields like `video_id`, `title`, `channel_title`, `category_id`, `views`, `likes`, `dislikes`, `comment_count`, `trending_date`, and more.

---

## Project Structure

```
YouTubeAnalyticsAndDataPipeline/
├── S3-Commands.sh                            #AWS CLI commands to upload raw data to S3
├── de-youtubeanalysis-lambda-json-parquet.py #Lambda function: JSON → Parquet
├── de-youtubeanalysis-cleaned-csv-to-parquet.py #Glue ETL job: CSV → Parquet
└── YouTubeAnalyticsDashboard.pdf             #Final QuickSight dashboard export
```

---

## Pipeline Walkthrough

### 1. Data Ingestion
Raw data is uploaded to S3 using the AWS CLI. CSVs are stored with Hive-style region partitioning (`region=ca/`, `region=us/`, etc.) and JSON reference files are loaded separately.

```bash
aws s3 cp USvideos.csv s3://de-youtubeanalysis-raw/youtube/raw_statistics/region=us/
aws s3 cp . s3://de-youtubeanalysis-raw/youtube/raw_statistics_reference_data/ --recursive --exclude "*" --include "*.json"
```

### 2. Lambda: JSON → Parquet
An S3-triggered Lambda function reads incoming JSON reference files using `awswrangler`, flattens nested fields with `pd.json_normalize`, and writes the output as Parquet to the cleaned S3 bucket. It also registers the table in the Glue Data Catalog automatically.

### 3. Glue ETL: CSV → Parquet
A Glue job reads the raw CSV statistics from the Glue catalog, applies column mappings, resolves type conflicts, drops null fields, and writes region-partitioned Parquet files to the cleaned S3 layer. Only CA, GB, and US regions are processed using predicate pushdown for efficiency.

### 4. Querying with Athena
Once the Glue Crawler catalogs the cleaned data, Athena is used to run SQL queries directly on the Parquet files — no loading required.

### 5. Visualization with QuickSight
The Athena tables are connected to Amazon QuickSight to build an interactive analytics dashboard. See `YouTubeAnalyticsDashboard.pdf` for the final output.

---

## Key Design Decisions

- **Hive-style partitioning by region** on S3 enables partition pruning and faster Athena queries
- **Predicate pushdown in Glue** (`region in ('ca','gb','us')`) reduces data scanned during ETL
- **awswrangler** simplifies Lambda-based S3 reads and Glue Catalog writes without boilerplate
- **Parquet format** throughout the cleaned layer for columnar storage and compression benefits

---

## Dashboard Highlights

The QuickSight dashboard (`YouTubeAnalyticsDashboard.pdf`) covers:
- Trending video counts by region
- Top channels by views, likes, and comment engagement
- Category-level performance breakdown
- Engagement ratios (likes/views, comments/views)

---

## Setup

### Prerequisites
- AWS account with IAM roles for S3, Lambda, Glue, Athena, and QuickSight
- AWS CLI configured locally
- Python 3.x

### Steps

1. Create raw and cleaned S3 buckets
2. Upload data using `S3-Commands.sh`
3. Set up Glue Crawlers on the raw bucket to catalog schemas
4. Deploy the Lambda function with the appropriate S3 trigger and environment variables:
   - `s3_cleansed_layer`
   - `glue_catalog_db_name`
   - `glue_catalog_table_name`
   - `write_data_operation`
5. Run the Glue ETL job for CSV processing
6. Run a Glue Crawler on the cleaned bucket
7. Query via Athena and connect to QuickSight

---

## Future Improvements

- Automate the full pipeline with AWS Step Functions or EventBridge
- Add data quality checks (e.g., Great Expectations or Glue Data Quality)
- Expand region coverage beyond CA, GB, and US
- Schedule incremental loads instead of full refreshes
