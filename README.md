# Redfin Data Engineering Pipeline

This project creates an automated data pipeline for extracting, transforming, and loading Redfin real estate market data. The pipeline extracts data from Redfin's public data sets, transforms it for analysis, and loads it into Snowflake via AWS S3.

## Architecture Overview

![Architecture Diagram](https://via.placeholder.com/800x400?text=Redfin+Data+Pipeline+Architecture)

The pipeline follows these steps:
1. **Extract**: Pull data from Redfin's public S3 bucket
2. **Transform**: Clean, filter, and enrich the data
3. **Load**: Store raw data in S3 and processed data in Snowflake
4. **Schedule**: Automate the pipeline using Apache Airflow

## Prerequisites

- AWS Account with S3 access
- EC2 instance (t2.medium or larger recommended)
- Snowflake account
- Python 3.8+

## Project Structure

```
Redfin_Data_Engineering/
├── Extract_and_Transform_WorkBook.ipynb  # Jupyter notebook for data exploration
├── README.md                             # This documentation file
├── RealEstate_script.txt                 # Snowflake SQL scripts
├── commands_run.txt                      # Setup commands for EC2
├── redfin_analytics.py                   # Airflow DAG definition
└── requirements.txt                      # Python dependencies
```

## Setup Instructions

### 1. Setting Up AWS EC2 Instance

1. Launch an EC2 instance (Ubuntu 20.04 LTS recommended)
2. Configure security groups to allow SSH access
3. Connect to your instance via SSH
4. Run the commands in `commands_run.txt` to set up the environment:

```bash
sudo apt update
sudo apt install python3-pip
sudo apt install python3.10-venv
python3 -m venv redfin_venv
source redfin_venv/bin/activate
pip install pandas
pip install boto3
pip install --upgrade awscli
pip install apache-airflow
```

5. Configure AWS CLI with your credentials:
```bash
aws configure
```

6. Start Airflow:
```bash
airflow standalone
```

### 2. S3 Bucket Setup

1. Create two S3 buckets:
   - `store-raw-data-yml`: for storing raw extracted data
   - `redfin-transform-zone-yml`: for storing transformed data

2. Set appropriate IAM permissions to allow EC2 to access these buckets

### 3. Snowflake Setup

1. Log in to your Snowflake account
2. Run the SQL commands in `RealEstate_script.txt` to create:
   - Database: `redfin_database_1`
   - Schema: `redfin_schema`
   - Table: `redfin_table`
   - File format: `format_csv`
   - External stage: `redfin_ext_stage_yml` (pointing to your S3 bucket)
   - Snowpipe: `redfin_snowpipe` (for continuous data ingestion)

3. Note: Replace AWS credentials in the script with your own:
```sql
CREATE OR REPLACE STAGE redfin_database_1.external_stage_schema.redfin_ext_stage_yml 
    url="s3://redfin-transform-zone-yml/"
    credentials=(aws_key_id='YOUR AWS KEY ID'
    aws_secret_key='YOUR AWS SECRET KEY')
    FILE_FORMAT = redfin_database_1.file_format_schema.format_csv;
```

### 4. Airflow DAG Setup

1. Copy `redfin_analytics.py` to the Airflow DAGs directory:
```bash
cp redfin_analytics.py ~/airflow/dags/
```

2. The DAG performs three main tasks:
   - `tsk_extract_redfin_data`: Extracts data from Redfin's public dataset
   - `tsk_transform_redfin_data`: Cleans and transforms the data
   - `tsk_load_to_s3`: Moves the raw data to S3 while transformed data is uploaded directly in the transform task

3. Verify the DAG is visible in the Airflow UI at http://localhost:8080

## File Descriptions

### Extract_and_Transform_WorkBook.ipynb

This Jupyter notebook demonstrates the ETL process for exploratory purposes:
- Shows how to extract data from Redfin's public S3 bucket
- Explores the data structure and sample records
- Demonstrates necessary transformations
- Provides visualization and analysis examples

### RealEstate_script.txt

Contains Snowflake SQL commands to:
- Create database, schema, and table structures
- Define file formats for CSV import
- Create external stage linking to S3
- Set up Snowpipe for automated data loading

### commands_run.txt

Lists all commands needed to set up the EC2 environment:
- System updates and Python installation
- Python virtual environment setup
- Required package installation
- AWS CLI and Airflow configuration

### redfin_analytics.py

Defines the Airflow DAG for automating the pipeline:
- Task definitions for extract, transform, and load operations
- S3 integration for data storage
- Scheduling parameters (configured for weekly runs)
- Error handling and retry logic

### requirements.txt

Comprehensive list of Python dependencies for reproducing the environment.

## Connecting Snowflake to EC2 and S3

### AWS S3 to Snowflake Integration

1. Create an IAM user with S3 access in AWS
2. Generate access keys for this user
3. In Snowflake, create a storage integration:

```sql
CREATE STORAGE INTEGRATION s3_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::account_number:role/role_name'
  STORAGE_ALLOWED_LOCATIONS = ('s3://redfin-transform-zone-yml/');
```

4. Retrieve the IAM user ARN from Snowflake:

```sql
DESC INTEGRATION s3_int;
```

5. Create a trust relationship in AWS IAM to allow Snowflake access

### EC2 to Snowflake Connection

1. Install Snowflake connector for Python:

```bash
pip install snowflake-connector-python
```

2. Add connection code to your Python scripts:

```python
import snowflake.connector

# Connect to Snowflake
conn = snowflake.connector.connect(
    user='YOUR_USERNAME',
    password='YOUR_PASSWORD',
    account='YOUR_ACCOUNT',
    warehouse='redfin_warehouse',
    database='redfin_database_1',
    schema='redfin_schema'
)

# Create a cursor
cursor = conn.cursor()

# Execute a query
cursor.execute("SELECT * FROM redfin_table LIMIT 10")
```

## Data Refresh Schedule

The pipeline is configured to run weekly, but you can adjust the schedule in the DAG:

```python
with DAG('redfin_analytics_dag',
        default_args=default_args,
        schedule_interval = '@weekly',  # Change as needed
        catchup=False) as dag:
```

Options include:
- `@daily` - Once every day
- `@weekly` - Once every week
- `@monthly` - Once every month
- Cron expressions for custom schedules

## Data Visualization

After successfully loading data into Snowflake and S3, you can create visualizations using industry-standard BI tools. This section outlines how to connect your data sources to PowerBI and Tableau for creating insightful dashboards.

### Visualizing with Power BI

#### Connecting to S3 Directly

1. **Install AWS Connector**: 
   - Open Power BI Desktop
   - Click "Get Data" → "More..." → Search for "Amazon S3"
   - You may need to download the Amazon S3 connector if not already installed

2. **Configure S3 Connection**:
   - Enter your AWS access key and secret key when prompted
   - Browse to your bucket (redfin-transform-zone-yml)
   - Select the CSV file you want to analyze

3. **Data Transformation**:
   - After importing, use Power Query Editor to:
     - Set correct data types (dates, numbers, etc.)
     - Rename columns for clarity
     - Create calculated columns if needed

4. **Building Visualizations**:
   - Create time series charts for median sale prices
   - Build maps showing property values by region
   - Develop comparative analyses across property types
   - Set up slicers for filtering by year, property type, and region

5. **Refreshing Data**:
   - Configure scheduled refreshes to keep dashboards up-to-date
   - Set up gateway connections for automated refreshes

#### Connecting via Snowflake

1. **Install Snowflake Connector**:
   - Click "Get Data" → "Database" → "Snowflake"

2. **Configure Connection**:
   - Server: your Snowflake account URL (e.g., `organization-account.snowflakecomputing.com`)
   - Warehouse: `redfin_warehouse`
   - Database: `redfin_database_1`
   - Schema: `redfin_schema`
   - Authentication: Username/Password or OAuth

3. **Import Data**:
   - Select the tables you want to analyze
   - Choose either import mode or DirectQuery depending on data size

### Visualizing with Tableau

#### Connecting to S3 Directly

1. **Install AWS Connector**:
   - If not already installed, download the Amazon S3 connector from Tableau's website
   - Open Tableau Desktop
   - Go to "Connect" → "To a Server" → "Amazon S3"

2. **Configure S3 Connection**:
   - Enter your AWS access key ID and secret access key
   - Select your S3 bucket (redfin-transform-zone-yml)
   - Choose the file format (CSV)
   - Select the specific file to analyze

3. **Data Preparation**:
   - In the Data Source tab:
     - Set geographic roles for location fields
     - Adjust data types as needed
     - Create calculated fields for analysis

4. **Building Visualizations**:
   - Create dashboards showing:
     - Price trends over time
     - Geographic heat maps of property values
     - Comparative analyses between cities and states
     - Inventory and sales metrics

5. **Publishing and Sharing**:
   - Publish to Tableau Server or Tableau Online
   - Set up scheduled refreshes to keep data current
   - Configure data source authentication

#### Connecting via Snowflake

1. **Use Built-in Connector**:
   - Go to "Connect" → "To a Server" → "Snowflake"

2. **Configure Connection**:
   - Server: your Snowflake server address
   - Username and Password
   - Warehouse: `redfin_warehouse`
   - Database: `redfin_database_1`
   - Schema: `redfin_schema`

3. **Import Data**:
   - Select the tables to analyze
   - Choose between live connection or extract based on performance needs

### Common Visualizations for Real Estate Analysis

1. **Time Series Analysis**:
   - Median sale price trends over time
   - Inventory levels by month/year
   - Seasonal patterns in sales activity

2. **Geographic Analysis**:
   - Heat maps of property values by city/state
   - Bubble charts showing sales volume by location
   - Comparative regional performance

3. **Market Performance Metrics**:
   - Days on market by property type
   - Sale-to-list price ratio analysis
   - Supply and demand indicators

4. **Comparative Analysis**:
   - Property type performance comparison
   - Year-over-year growth rates
   - City/state ranking dashboards

### Best Practices for Real Estate Dashboards

1. **Use Effective Filters**:
   - Enable filtering by:
     - Time period (year, quarter, month)
     - Property type
     - Geographic region
     - Price range

2. **Incorporate Geographic Visualization**:
   - Use maps to show spatial patterns
   - Enable drill-down from state to city level
   - Color-code regions by key metrics

3. **Show Historical Context**:
   - Include year-over-year comparisons
   - Display long-term trends alongside current data
   - Highlight seasonal patterns

4. **Optimize for Different Users**:
   - Create executive summary dashboards
   - Develop detailed analyst views
   - Design operational dashboards for day-to-day monitoring

## Troubleshooting

- **Airflow DAG not running**: Check logs in Airflow UI
- **S3 access issues**: Verify IAM permissions
- **Snowpipe failures**: Check integration status in Snowflake

## Future Enhancements

- Add data quality checks
- Set up alerting for pipeline failures
- Add more data sources for comprehensive analysis
