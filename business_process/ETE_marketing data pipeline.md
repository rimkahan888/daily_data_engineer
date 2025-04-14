

# End-to-End Marketing Data Pipeline Example

I'd be happy to share a detailed example of building an end-to-end data pipeline for a marketing department at a SaaS company using AWS, Airflow, Snowflake, and dbt.

## Project Background

At MarketBoost (a B2B SaaS company offering marketing automation tools), I led the development of a comprehensive marketing analytics pipeline. The marketing team needed to consolidate data from multiple sources to better understand customer acquisition costs, campaign performance, and customer journey analytics.

## Business Requirements

The marketing department needed:
- Daily consolidated view of campaign performance across channels
- Customer attribution modeling across touchpoints
- Cohort analysis for retention metrics
- Automated reporting for executives
- Self-service analytics for the marketing team

## Data Sources

We integrated data from:
1. Salesforce (customer and opportunity data)
2. HubSpot (marketing automation and email campaigns)
3. Google Analytics and Google Ads
4. Facebook Ads
5. LinkedIn Marketing
6. Customer product usage data from our application database
7. Stripe payment data

## Architecture Overview

```
[Data Sources] → [AWS S3] → [Airflow] → [Snowflake Raw] → [dbt] → [Snowflake Warehouse] → [Tableau/Looker]
```

## Detailed Implementation

### Ingestion Layer (AWS & Airflow)

I built custom Airflow DAGs hosted on AWS ECS to extract data:

```python
# Example DAG for Facebook Ads extraction
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from datetime import datetime, timedelta
import facebook_business_sdk as fb
import boto3
import json

default_args = {
    'owner': 'data_engineering',
    'depends_on_past': False,
    'start_date': datetime(2023, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5)
}

def extract_facebook_ads_data(**kwargs):
    # Auth with Facebook API
    app_id = "{{{{ conn.facebook_ads.login }}}}"
    app_secret = "{{{{ conn.facebook_ads.password }}}}"
    access_token = "{{{{ conn.facebook_ads.extra_dejson.access_token }}}}"
    account_id = "{{{{ conn.facebook_ads.extra_dejson.account_id }}}}"
    
    # Extract campaign data for date range
    execution_date = kwargs['execution_date']
    date_str = execution_date.strftime('%Y-%m-%d')
    
    # Facebook API client setup
    api = fb.APIClient(app_id, app_secret, access_token)
    account = fb.AdAccount(f'act_{account_id}')
    
    # Extract campaign data
    campaigns = account.get_campaigns(fields=[
        'id', 'name', 'status', 'objective', 
        'spend', 'impressions', 'clicks', 'conversions'
    ], params={
        'time_range': {'since': date_str, 'until': date_str},
        'level': 'campaign'
    })
    
    # Convert to JSON and upload to S3
    s3_client = boto3.client('s3')
    s3_client.put_object(
        Bucket='marketboost-data-lake',
        Key=f'raw/facebook_ads/campaigns/{date_str}/campaigns.json',
        Body=json.dumps([c.export_data() for c in campaigns])
    )
    
    # Additional extractions for ads, adsets, etc.

with DAG('facebook_ads_extraction', default_args=default_args, schedule_interval='@daily') as dag:
    extract_facebook_ads = PythonOperator(
        task_id='extract_facebook_ads_data',
        python_callable=extract_facebook_ads_data,
        provide_context=True
    )
    
    # Additional tasks for validation, notifications, etc.
```

For each data source, we created separate DAGs with appropriate error handling, logging, and monitoring. We used Airflow connections to securely store credentials and AWS Secrets Manager for sensitive information.

### Storage & Processing Layer (AWS S3, Snowflake)

Raw data was stored in S3 in a partitioned structure:
```
s3://marketboost-data-lake/
    ├── raw/
    │   ├── facebook_ads/campaigns/YYYY-MM-DD/
    │   ├── google_ads/campaigns/YYYY-MM-DD/
    │   ├── hubspot/contacts/YYYY-MM-DD/
    │   └── ...
    └── processed/
        └── ...
```

Then we used Snowflake's native S3 integration to load this data:

```sql
-- Snowflake external stage setup
CREATE OR REPLACE STAGE facebook_ads_stage
  URL = 's3://marketboost-data-lake/raw/facebook_ads/'
  CREDENTIALS = (AWS_KEY_ID = '{{aws_access_key_id}}' AWS_SECRET_KEY = '{{aws_secret_access_key}}');

-- File format for JSON
CREATE OR REPLACE FILE FORMAT json_format
  TYPE = 'JSON' 
  STRIP_OUTER_ARRAY = TRUE;

-- Raw table for Facebook Ads campaigns
CREATE OR REPLACE TABLE raw.facebook_ads_campaigns (
  raw_data VARIANT,
  load_timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  source_file VARCHAR
);

-- Copy command executed by Airflow
COPY INTO raw.facebook_ads_campaigns (raw_data, source_file)
FROM (
  SELECT $1, METADATA$FILENAME
  FROM @facebook_ads_stage/campaigns/
)
FILE_FORMAT = json_format
PATTERN = '.*/.*/campaigns.json'
ON_ERROR = 'CONTINUE';
```

### Transformation Layer (dbt)

We used dbt for transforming raw data into analytics-ready models:

```yaml
# models/marketing/schema.yml
version: 2

models:
  - name: stg_facebook_ads_campaigns
    description: Standardized Facebook Ads campaign data
    columns:
      - name: campaign_id
        description: Unique identifier for the campaign
        tests:
          - unique
          - not_null
      - name: campaign_name
        description: Name of the campaign
      - name: date
        description: Date of the campaign metrics
        tests:
          - not_null
      # More column definitions...
```

```sql
-- models/marketing/staging/stg_facebook_ads_campaigns.sql
WITH source AS (
    SELECT
        raw_data,
        load_timestamp,
        source_file
    FROM {{ source('raw', 'facebook_ads_campaigns') }}
)

SELECT
    raw_data:id::VARCHAR AS campaign_id,
    raw_data:name::VARCHAR AS campaign_name,
    raw_data:status::VARCHAR AS campaign_status,
    raw_data:objective::VARCHAR AS campaign_objective,
    raw_data:spend::FLOAT AS campaign_spend,
    raw_data:impressions::INTEGER AS impressions,
    raw_data:clicks::INTEGER AS clicks,
    TO_DATE(REGEXP_SUBSTR(source_file, '[0-9]{4}-[0-9]{2}-[0-9]{2}')) AS date,
    ROUND(raw_data:clicks::FLOAT / NULLIF(raw_data:impressions::FLOAT, 0), 4) AS ctr,
    ROUND(raw_data:spend::FLOAT / NULLIF(raw_data:clicks::FLOAT, 0), 2) AS cpc,
    load_timestamp,
    'facebook_ads' AS source
FROM source
```

For the cross-channel marketing attribution model:

```sql
-- models/marketing/marts/mart_marketing_attribution.sql
WITH touchpoints AS (
    SELECT
        user_id,
        touchpoint_timestamp,
        touchpoint_source,
        touchpoint_medium,
        touchpoint_campaign,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY touchpoint_timestamp) AS touchpoint_order
    FROM {{ ref('int_user_marketing_touchpoints') }}
),

conversions AS (
    SELECT
        user_id,
        conversion_timestamp,
        conversion_value,
        conversion_type
    FROM {{ ref('int_user_conversions') }}
),

attributed_conversions AS (
    SELECT
        c.user_id,
        c.conversion_timestamp,
        c.conversion_value,
        c.conversion_type,
        -- First-touch attribution
        FIRST_VALUE(t.touchpoint_source) OVER (
            PARTITION BY c.user_id, c.conversion_timestamp 
            ORDER BY t.touchpoint_timestamp
        ) AS first_touch_source,
        -- Last-touch attribution
        LAST_VALUE(t.touchpoint_source) OVER (
            PARTITION BY c.user_id, c.conversion_timestamp 
            ORDER BY t.touchpoint_timestamp
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS last_touch_source,
        -- Multi-touch attribution (linear model)
        COUNT(DISTINCT t.touchpoint_source) OVER (
            PARTITION BY c.user_id, c.conversion_timestamp
        ) AS total_touchpoints
    FROM conversions c
    JOIN touchpoints t 
        ON c.user_id = t.user_id 
        AND t.touchpoint_timestamp <= c.conversion_timestamp
        AND t.touchpoint_timestamp >= DATEADD(day, -30, c.conversion_timestamp)
)

SELECT
    user_id,
    conversion_timestamp,
    conversion_value,
    conversion_type,
    first_touch_source,
    last_touch_source,
    total_touchpoints,
    -- Linear attribution model
    ROUND(conversion_value / total_touchpoints, 2) AS attributed_value_per_touchpoint
FROM attributed_conversions
```

### Orchestration

We used Airflow as the central orchestration tool, creating a master DAG that ensured proper execution order:

```python
# Master marketing pipeline DAG
from airflow import DAG
from airflow.operators.dummy_operator import DummyOperator
from airflow.sensors.external_task_sensor import ExternalTaskSensor
from airflow.contrib.operators.snowflake_operator import SnowflakeOperator
from airflow.operators.bash_operator import BashOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data_engineering',
    'depends_on_past': False,
    'start_date': datetime(2023, 1, 1),
    'email_on_failure': True,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'marketing_master_pipeline',
    default_args=default_args,
    schedule_interval='0 5 * * *',  # Daily at 5 AM
    catchup=False
) as dag:
    
    start = DummyOperator(task_id='start')
    
    # Wait for all extract DAGs to complete
    wait_for_facebook = ExternalTaskSensor(
        task_id='wait_for_facebook_extract',
        external_dag_id='facebook_ads_extraction',
        external_task_id=None,
        execution_delta=timedelta(hours=0),
        timeout=3600
    )
    
    wait_for_google = ExternalTaskSensor(
        task_id='wait_for_google_extract',
        external_dag_id='google_ads_extraction',
        external_task_id=None,
        execution_delta=timedelta(hours=0),
        timeout=3600
    )
    
    # Additional wait sensors for other sources...
    
    # Load data to Snowflake
    load_to_snowflake = SnowflakeOperator(
        task_id='load_to_snowflake',
        sql='CALL marketing_schema.sp_load_marketing_data();',
        snowflake_conn_id='snowflake_conn'
    )
    
    # Run dbt transformations
    run_dbt = BashOperator(
        task_id='run_dbt',
        bash_command='cd /opt/airflow/dbt && dbt run --models tag:marketing --profiles-dir .',
        env={
            'DBT_SNOWFLAKE_ACCOUNT': '{{ conn.snowflake_conn.account }}',
            'DBT_SNOWFLAKE_USER': '{{ conn.snowflake_conn.login }}',
            'DBT_SNOWFLAKE_PASSWORD': '{{ conn.snowflake_conn.password }}',
            'DBT_SNOWFLAKE_ROLE': 'TRANSFORM_ROLE',
            'DBT_SNOWFLAKE_DATABASE': 'ANALYTICS',
            'DBT_SNOWFLAKE_WAREHOUSE': 'TRANSFORM_WH'
        }
    )
    
    # Run tests
    run_dbt_tests = BashOperator(
        task_id='run_dbt_tests',
        bash_command='cd /opt/airflow/dbt && dbt test --models tag:marketing --profiles-dir .',
        env={
            # Same environment variables as above
        }
    )
    
    # Signal completion for downstream processes
    end = DummyOperator(task_id='end')
    
    # Set task dependencies
    start >> [wait_for_facebook, wait_for_google] >> load_to_snowflake >> run_dbt >> run_dbt_tests >> end
```

### Deployment & Infrastructure

We used AWS CDK to define our infrastructure as code:

1. **ECS Fargate cluster** for Airflow workers
2. **RDS PostgreSQL** for Airflow metadata database
3. **S3 buckets** for data lake storage
4. **IAM roles** with least-privilege permissions
5. **AWS CloudWatch** for monitoring and alerting

For Snowflake infrastructure:

```sql
-- Warehouse setup
CREATE WAREHOUSE IF NOT EXISTS TRANSFORM_WH 
WITH WAREHOUSE_SIZE = 'MEDIUM' 
AUTO_SUSPEND = 300
AUTO_RESUME = TRUE;

-- Database and schema setup
CREATE DATABASE IF NOT EXISTS ANALYTICS;

CREATE SCHEMA IF NOT EXISTS ANALYTICS.RAW;
CREATE SCHEMA IF NOT EXISTS ANALYTICS.STAGING;
CREATE SCHEMA IF NOT EXISTS ANALYTICS.INTERMEDIATE;
CREATE SCHEMA IF NOT EXISTS ANALYTICS.MARTS;

-- Role-based access control
CREATE ROLE IF NOT EXISTS TRANSFORM_ROLE;
GRANT USAGE ON WAREHOUSE TRANSFORM_WH TO ROLE TRANSFORM_ROLE;
GRANT ALL ON DATABASE ANALYTICS TO ROLE TRANSFORM_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ANALYTICS TO ROLE TRANSFORM_ROLE;
GRANT ALL ON FUTURE TABLES IN SCHEMA ANALYTICS.RAW TO ROLE TRANSFORM_ROLE;
-- Additional grants...
```

## Business Impact

This pipeline delivered significant value:
1. Reduced marketing data processing time from days to hours
2. Improved attribution modeling led to 22% better ROI on ad spend
3. Enabled self-service analytics for the marketing team
4. Automated daily executive dashboards
5. Provided accurate cohort analysis for retention strategies

The marketing team gained visibility into their multi-channel campaigns, allowing them to optimize spend across channels based on actual performance data rather than siloed metrics.

## Technical Challenges & Solutions

1. **Data Volume**: Some sources generated over 50GB of daily data. We implemented partition pruning in Snowflake and parallelized Airflow tasks to handle this efficiently.

2. **Data Quality**: We implemented dbt tests and data quality checks at each stage, with alerts for anomalies.

3. **API Rate Limits**: Implemented backoff strategies and batch processing in Airflow tasks to respect API limits.

4. **Cost Management**: Used Snowflake's auto-suspend feature and implemented task-specific warehouses to optimize cost.

This end-to-end marketing data pipeline became a central component of the company's data strategy, laying the groundwork for advanced analytics and machine learning initiatives.
