📦 Resilient Data Ingestion Pipeline (S3 → RDS with Glue Fallback) using Docker (Shell Script)
📌 Project Overview

This project implements a fault-tolerant data ingestion pipeline using a Shell Script, where data is read from S3 and inserted into RDS.

If the database is unavailable, the system automatically falls back to AWS Glue, ensuring reliability and zero data loss.

🎯 Objective
Read CSV data from S3
Insert data into RDS (MySQL-compatible)
Automatically fallback to Glue if RDS fails
Containerize using Docker
Build a lightweight, script-based ingestion system
🏗️ Architecture Overview

AWS Services Used:

Amazon S3 → Data source
Amazon RDS → Primary database
AWS Glue → Fallback system
Docker → Containerization
🔄 Data Flow
S3 (CSV File)
     ↓
Shell Script (.sh)
     ↓
Try → RDS (MySQL CLI)
     ↓
If Failure ❌
     ↓
Fallback → AWS Glue (Catalog Table)
📂 Project Structure
.
├── ingest.sh             # Main shell script
├── Dockerfile            # Docker configuration
├── config.env            # Environment variables
└── README.md
⚙️ Shell Script Functionality
📌 Features:
Downloads CSV from S3 using AWS CLI
Loads data into RDS using MySQL CLI
Detects failure using exit status ($?)
Falls back to AWS Glue using AWS CLI
Logs execution steps
🧾 Sample Shell Script (ingest.sh)
#!/bin/bash

set -e

echo "Starting Data Ingestion..."

# Load config
source config.env

# Step 1: Download file from S3
echo "Downloading file from S3..."
aws s3 cp s3://$S3_BUCKET/$S3_KEY /tmp/data.csv

# Step 2: Try inserting into RDS
echo "Uploading data to RDS..."

mysql -h $RDS_HOST -u $RDS_USER -p$RDS_PASSWORD $RDS_DB <<EOF
LOAD DATA LOCAL INFILE '/tmp/data.csv'
INTO TABLE $RDS_TABLE
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
EOF

# Check status
if [ $? -eq 0 ]; then
    echo "Data inserted into RDS successfully ✅"
else
    echo "RDS failed ❌ Falling back to Glue..."

    # Step 3: Upload file for Glue usage
    aws s3 cp /tmp/data.csv $GLUE_S3_PATH

    # Step 4: Create Glue Table
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

    echo "Glue fallback executed successfully ✅"
fi
🐳 Docker Setup
📌 Dockerfile
FROM amazonlinux:2

RUN yum install -y mysql aws-cli

WORKDIR /app

COPY ingest.sh .
COPY config.env .

RUN chmod +x ingest.sh

CMD ["./ingest.sh"]
⚙️ Configuration (config.env)
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
▶️ Build & Run
Build Docker Image
docker build -t ingestion-shell-app .
Run Container
docker run --env-file config.env ingestion-shell-app
📊 Expected Output
✅ RDS Success
Starting Data Ingestion...
Downloading file from S3...
Uploading data to RDS...
Data inserted into RDS successfully
❌ RDS Failure → Glue
RDS failed Falling back to Glue...
Glue fallback executed successfully
📸 Deliverables
✅ Shell script (ingest.sh)
✅ Dockerfile
✅ Logs showing:
RDS success OR
Glue fallback
✅ Screenshot of:
RDS table OR
Glue table
🧠 Challenges & Solutions
❗ Challenge: Handling DB Failure

Solution: Used exit status ($?) for fallback logic

❗ Challenge: AWS CLI in Docker

Solution: Used Amazon Linux base image

🔐 Security Considerations
Avoid hardcoding credentials
Use IAM roles instead of access keys
Restrict S3 and RDS permissions
Secure config file:
chmod 600 config.env
🚀 Key Learnings
Shell-based data pipelines
AWS CLI automation
Failure handling using bash
Dockerizing scripts
Real-world DevOps + Data Engineering integration
🔮 Future Enhancements
Add retry before fallback
Use AWS Lambda for event-driven execution
Add logging to Amazon CloudWatch
Schema auto-detection for Glue
🔥 GitHub Description

Shell script–based data ingestion pipeline that transfers data from S3 to RDS with automatic fallback to AWS Glue, fully containerized using Docker.

🏷️ GitHub Topics
aws
amazon-s3
amazon-rds
aws-glue
bash
shell-script
docker
data-pipeline
etl
devops
cloud-computing
