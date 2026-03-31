# Resilient Data Ingestion Pipeline (S3 to RDS with Glue Fallback) using Docker

## Project Overview

This project implements a fault-tolerant data ingestion pipeline using a shell script. The pipeline reads CSV data from Amazon S3 and attempts to insert it into an Amazon RDS database. If the database is unavailable or the insertion fails, the system automatically falls back to AWS Glue to ensure that data is not lost.

The solution is containerized using Docker, making it portable and easy to deploy across environments.

## Objective

* Read CSV data from Amazon S3
* Insert data into Amazon RDS (MySQL-compatible)
* Automatically fall back to AWS Glue in case of failure
* Containerize the application using Docker
* Build a lightweight and reliable ingestion system

## Architecture Overview

### Services Used

* Amazon S3 (Data Source)
* Amazon RDS (Primary Database)
* AWS Glue (Fallback System)
* Docker (Containerization)

## Data Flow

1. Data is stored in Amazon S3 as a CSV file
2. A shell script downloads the file
3. The script attempts to insert data into Amazon RDS
4. If the insertion fails, the script uploads the data to S3 and registers it in AWS Glue

## Project Structure

```
.
├── ingest.sh
├── Dockerfile
├── config.env
└── README.md
```

## Shell Script Functionality

### Features

* Downloads CSV file from S3 using AWS CLI
* Loads data into RDS using MySQL CLI
* Detects failure using exit status
* Falls back to AWS Glue using AWS CLI
* Logs execution steps

## Shell Script (ingest.sh)

```bash
#!/bin/bash

set -e

echo "Starting Data Ingestion..."

# Load configuration
source config.env

# Step 1: Download file from S3
echo "Downloading file from S3..."
aws s3 cp s3://$S3_BUCKET/$S3_KEY /tmp/data.csv

# Step 2: Insert into RDS
echo "Uploading data to RDS..."

mysql -h $RDS_HOST -u $RDS_USER -p$RDS_PASSWORD $RDS_DB <<EOF
LOAD DATA LOCAL INFILE '/tmp/data.csv'
INTO TABLE $RDS_TABLE
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
EOF

# Step 3: Check status and fallback
if [ $? -eq 0 ]; then
    echo "Data inserted into RDS successfully"
else
    echo "RDS failed. Falling back to Glue"

    aws s3 cp /tmp/data.csv $GLUE_S3_PATH

    aws glue create-table --database-name $GLUE_DB --table-input '{
        "Name": "'$GLUE_TABLE'",
        "StorageDescriptor": {
            "Columns": [
                {"Name": "col1", "Type": "string"}
            ],
            "Location": "'$GLUE_S3_PATH'",
            "InputFormat": "org.apache.hadoop.mapred.TextInputFormat",
            "OutputFormat": "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat"
        },
        "TableType": "EXTERNAL_TABLE"
    }'

    echo "Glue fallback executed successfully"
fi
```

## Docker Configuration

### Dockerfile

```
FROM amazonlinux:2

RUN yum install -y mysql aws-cli

WORKDIR /app

COPY ingest.sh .
COPY config.env .

RUN chmod +x ingest.sh

CMD ["./ingest.sh"]
```

## Configuration File (config.env)

```
S3_BUCKET=your-bucket-name
S3_KEY=data/sample.csv

RDS_HOST=your-rds-endpoint
RDS_USER=admin
RDS_PASSWORD=password
RDS_DB=yourdb
RDS_TABLE=yourtable

GLUE_DB=your_glue_db
GLUE_TABLE=your_glue_table
GLUE_S3_PATH=s3://your-bucket/glue-data/
```

## Build and Run

### Build Docker Image

```
docker build -t ingestion-shell-app .
```

### Run Container

```
docker run --env-file config.env ingestion-shell-app
```

## Expected Output

### Successful RDS Ingestion

```
Starting Data Ingestion...
Downloading file from S3...
Uploading data to RDS...
Data inserted into RDS successfully
```

### RDS Failure with Glue Fallback

```
Starting Data Ingestion...
Downloading file from S3...
Uploading data to RDS...
RDS failed. Falling back to Glue
Glue fallback executed successfully
```

## Verification

* Verify data in RDS table after successful ingestion
* Verify data in S3 and Glue table after fallback
* Check container logs for execution details

## Challenges and Solutions

**Challenge:** Handling database failures
**Solution:** Used exit status to trigger fallback mechanism

**Challenge:** Running AWS CLI inside container
**Solution:** Used Amazon Linux base image with required tools installed

## Security Considerations

* Avoid hardcoding credentials in scripts
* Use IAM roles instead of access keys
* Restrict permissions for S3, RDS, and Glue
* Secure configuration file using appropriate file permissions

## Optional Enhancements

* Add retry mechanism before fallback
* Use AWS Lambda for event-driven execution
* Integrate logging with CloudWatch
* Implement schema detection for Glue

## Key Learnings

* Building shell-based data pipelines
* Automating workflows using AWS CLI
* Handling failures in data ingestion systems
* Containerizing applications using Docker
* Integrating DevOps and data engineering practices

## Conclusion

This project demonstrates a reliable and fault-tolerant approach to data ingestion using AWS services. It ensures that data is preserved even in failure scenarios and provides a flexible, containerized solution suitable for real-world applications.
