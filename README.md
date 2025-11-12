# End-to-End AWS Data Engineering ETL Pipeline

## Objective
This project demonstrates a serverless, end-to-end ETL (Extract, Transform, Load) pipeline on AWS. The pipeline automatically processes semi-structured JSON data, flattens it into a structured columnar format (Parquet), catalogs it for easy discovery, and makes it available for querying using standard SQL.

## Architecture
The project uses the following AWS services to create a fully automated, event-driven data pipeline:

1.  **Amazon S3:** Used as the central data lake for storing raw JSON files and processed Parquet files.
2.  **AWS Lambda:** Serves as the compute layer to transform (flatten) JSON data into Parquet format upon arrival.
3.  **AWS Glue Crawler:** Automatically catalogs the data in the S3 data lake, inferring the schema and creating/updating tables in the Data Catalog.
4.  **Amazon Athena:** Allows us to run SQL queries directly on the data stored in S3 using the schema defined in the Glue Data Catalog.

### Data Flow
1.  A new JSON file is uploaded to the `myJSON/` folder in S3.
2.  This event automatically triggers an AWS Lambda function.
3.  The Lambda function reads the JSON file, flattens its nested structure, and converts it into a Parquet file.
4.  The new Parquet file is saved to the `orders_parquet_datalake/` folder in S3.
5.  The Lambda function then triggers an AWS Glue Crawler.
6.  The Glue Crawler scans the `orders_parquet_datalake/` folder, infers the schema, and creates a table in the Glue Data Catalog.
7.  The data is now ready to be queried using Amazon Athena.

## Step-by-Step Implementation Guide

### Prerequisites
- An active AWS account.
- Basic familiarity with the AWS Management Console (S3, Lambda, IAM, Glue).
- A sample JSON file (see `sample_data.json` in this repo for the expected format).

### Step 1: Set up the S3 Bucket and Folder Structure
1.  Navigate to the **Amazon S3** console and create a new bucket.
2.  Inside your bucket, create two folders:
    - `myJSON/` - This will be the staging area for incoming raw JSON files.
    - `orders_parquet_datalake/` - This will store the processed Parquet files.

### Step 2: Create the IAM Role for Lambda
The Lambda function needs permissions to access other AWS services.
1.  Go to **IAM > Roles** and click "Create role".
2.  Select "AWS service" as the trusted entity and choose "Lambda".
3.  Attach the following managed policies:
    - `AmazonS3FullAccess`
    - `AWSGlueServiceRole`
    - `AWSLambdaBasicExecutionRole` (for CloudWatch logs)
4.  Name the role (e.g., `lambda-s3-glue-execution-role`) and create it.

### Step 3: Create and Deploy the Lambda Function
1.  Navigate to **AWS Lambda** and click "Create function".
2.  Choose "Author from scratch".
3.  Provide a name (e.g., `json-flatten-to-parquet`).
4.  Set the **Runtime** to `Python 3.9` or higher.
5.  Under "Permissions", choose "Use an existing role" and select the IAM role you created in Step 2.
6.  Click "Create function".
7.  In the function code section, copy and paste the code from the `lambda_function.py` file provided in this repository.
8.  Click "Deploy" to save your function.

**Important:** Since the Lambda uses `pandas` and `pyarrow`, you must add these as a layer.
- You can use a public layer like `arn:aws:lambda:us-east-1:336392948345:layer:AWSSDKPandas-Python39:7` (replace `us-east-1` with your region). Add this via the "Layers" section in the Lambda console.

### Step 4: Configure the S3 Trigger for Lambda
1.  In your Lambda function, go to the "Configuration" tab and click "Add trigger".
2.  Select "S3" as the source.
3.  Choose the S3 bucket you created.
4.  Set the **Event type** to "All object create events".
5.  For the **Prefix**, enter `myJSON/`. This ensures the Lambda is only triggered by files added to this specific folder.
6.  Click "Add".

### Step 5: Set up the AWS Glue Crawler
1.  Navigate to **AWS Glue > Crawlers** and click "Add crawler".
2.  **Name:** `etl_pipeline_crawler`
3.  **Data source:** Choose "S3" and select the path to your `orders_parquet_datalake/` folder.
4.  **IAM role:** Create a new role or select an existing one with permissions for Glue (e.g., `AWSGlueServiceRole`).
5.  **Target Database:** Create a new database (e.g., `orders_db`).
6.  **Table prefix:** (Optional) You can set a prefix like `parquet_` or leave it blank.
7.  Review and finish creating the crawler. **Run the crawler once manually** to create the initial table.

### Step 6: Test the End-to-End Pipeline
1.  Upload the provided `sample_data.json` file to your S3 bucket's `myJSON/` folder.
2.  This should automatically trigger your Lambda function. You can check the Lambda's **Monitor** tab and CloudWatch logs to see the execution details.
3.  Shortly after the Lambda finishes, a new Parquet file should appear in the `orders_parquet_datalake/` folder.
4.  Go to the Glue Console and run your `etl_pipeline_crawler` again. It will now discover the new Parquet file and update the table schema.
5.  Finally, go to **Amazon Athena**.
    - In the query editor, select your database (`orders_db`).
    - Run a query to see your data:
      ```sql
      SELECT * FROM "AwsDataCatalog"."orders_db"."parquet_orders_parquet_datalake" limit 10;
      ```
    (Replace `parquet_orders_parquet_datalake` with your actual table name).

## Repository Files
- `lambda_function.py`: The source code for the AWS Lambda function that performs the ETL.
- `sample_data.json`: A sample input file with the expected nested JSON structure.

## Future Enhancements
- Add error handling and retry logic for more robust production use.
- Partition the Parquet files by a date field (e.g., `order_date`) for cost-effective querying.
- Integrate with a visualization tool like Amazon QuickSight.
- Use Step Functions for more complex orchestration.
