# Web-Server-Log-Analysis
📌 Project Overview
This project analyzes web server logs using Apache Hive. The goal is to extract insights such as:

Total Web Requests
Status Code Analysis (200, 404, 500)
Top 3 Most Visited Pages
Traffic Source Analysis (User Agents)
Detect Suspicious Activity (IPs with failed requests)
Traffic Trends Over Time
Partitioning by Status Code for Optimization

Dataset Format (CSV)
The web server log file contains the following fields:

Field Name	Data Type	Description
ip	STRING	Client IP Address
timestamp	STRING	Request Time (YYYY-MM-DD HH:MM:SS)
url	STRING	Requested URL
status	INT	HTTP Status Code (200, 404, 500, etc.)
user_agent	STRING	Browser/User Agent

192.168.1.1,2024-02-01 10:15:00,/home,200,Mozilla/5.0
192.168.1.2,2024-02-01 10:16:00,/products,200,Chrome/90.0
192.168.1.3,2024-02-01 10:17:00,/checkout,404,Safari/13.1
192.168.1.10,2024-02-01 10:18:00,/home,500,Mozilla/5.0
192.168.1.15,2024-02-01 10:19:00,/products,404,Chrome/90.0

🚀 Implementation Steps
1️⃣ Setup Apache Hive with Docker
Create a docker-compose.yml file and start the services:
docker-compose up -d

2️⃣ Access Hive CLI
Enter the Hive shell
docker exec -it hive-server /bin/bash
hive
3️⃣ Create Database and Table:

CREATE DATABASE web_log_analysis;
USE web_log_analysis;

CREATE EXTERNAL TABLE web_logs (
    ip STRING,
    log_timestamp STRING,  -- Renamed "timestamp" (reserved keyword)
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE 
LOCATION '/user/hive/web_logs/';
4️⃣ Upload Data to HDFS
First, copy the CSV file into the Namenode container:

docker cp web_logs.csv namenode:/tmp/web_logs.csv
Move it into HDFS:

docker exec -it namenode /bin/bash
hdfs dfs -mkdir -p /user/hive/web_logs
hdfs dfs -put /tmp/web_logs.csv /user/hive/web_logs/

5️⃣ Load Data into Hive Table

LOAD DATA INPATH '/user/hive/web_logs/web_logs.csv' INTO TABLE web_logs;
📊 Querying the Data
🔹 Count Total Web Requests
SELECT COUNT(*) AS total_requests FROM web_logs;
🔹 Status Code Analysis
SELECT status, COUNT(*) AS count FROM web_logs GROUP BY status ORDER BY count DESC;
🔹 Top 3 Most Visited Pages
SELECT url, COUNT(*) AS visit_count FROM web_logs GROUP BY url ORDER BY visit_count DESC LIMIT 3;
🔹 Traffic Source Analysis (Most Common User Agents)
SELECT user_agent, COUNT(*) AS count FROM web_logs GROUP BY user_agent ORDER BY count DESC LIMIT 3;
🔹 Detect Suspicious IPs (More Than 3 Failed Requests)
SELECT ip, COUNT(*) AS failed_requests FROM web_logs WHERE status IN (404, 500) GROUP BY ip HAVING COUNT(*) > 3;
🔹 Traffic Trends (Requests per Minute)
SELECT SUBSTR(log_timestamp, 1, 16) AS minute, COUNT(*) AS request_count FROM web_logs GROUP BY SUBSTR(log_timestamp, 1, 16) ORDER BY minute;
🗂️ Partitioning for Optimization
Partitioning by status code improves query performance.

1️⃣ Create Partitioned Table
CREATE TABLE web_logs_partitioned (
    ip STRING,
    log_timestamp STRING,
    url STRING,
    user_agent STRING
)
PARTITIONED BY (status INT)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE;
2️⃣ Enable Dynamic Partitioning
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;
3️⃣ Insert Data into Partitioned Table
INSERT OVERWRITE TABLE web_logs_partitioned PARTITION(status)
SELECT ip, log_timestamp, url, user_agent, status FROM web_logs;
📝 Challenges Faced
Reserved Keyword Issue
"timestamp" is a Hive reserved keyword → Renamed to log_timestamp.
HDFS File Not Found Error
Ensured file was uploaded to /user/hive/web_logs/ before loading into Hive.
Slow Queries
Used Partitioning on status to optimize query performance.
📌 Sample Output
1️⃣ Total Web Requests
Total Requests: 100
2️⃣ Status Code Analysis
makefile
200: 80
404: 10
500: 10
3️⃣ Most Visited Pages
/home: 50
/products: 30
/checkout: 20
4️⃣ Suspicious IPs
makefile
192.168.1.10: 5 failed requests
192.168.1.15: 4 failed requests




