IoT Intrusion Detection Streaming Pipeline Using Real-Time Analytics
🚀 Step 3: Start Services docker-compose up -d

📡 Create Kafka Topic

docker exec kafka /opt/kafka/bin/kafka-topics.sh
--create --topic iot-topic
--bootstrap-server localhost:9092
--partitions 1 --replication-factor 1

📥 Run Kafka Consumer (Listening Mode) docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh
--topic iot-topic
--bootstrap-server localhost:9092
--from-beginning

🐍 Python Environment Setup

📦 Method 1: Virtual Environment (venv) python -m venv venv

Activate:

Windows

venv\Scripts\activate

Linux / Mac

source venv/bin/activate

Install dependencies:

pip install -r requirements.txt 🐍 Method 2: Conda Environment

Create environment:

conda create -n iot-env python=3.10

Activate:

conda activate iot-env

Install dependencies:

pip install -r requirements.txt

🐳 Verify Running Containers

docker ps

⚡ Open Flink SQL Client

docker exec -it flink-jobmanager /opt/flink/bin/sql-client.sh

🔌 Fix: Kafka Connector (IMPORTANT)

If Kafka table is not detected, install connector manually.

1️⃣ Download Kafka Connector

docker exec flink-jobmanager wget -P /opt/flink/lib
https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.4.0-1.20/flink-sql-connector-kafka-3.4.0-1.20.jar

docker exec flink-taskmanager wget -P /opt/flink/lib
https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.4.0-1.20/flink-sql-connector-kafka-3.4.0-1.20.jar

2️⃣ Restart Flink docker restart flink-taskmanager docker restart flink-jobmanager

Wait 10–15 seconds ⏳

3️⃣ Verify Installation docker exec flink-jobmanager ls /opt/flink/lib docker exec flink-taskmanager ls /opt/flink/lib

Expected:

flink-sql-connector-kafka-3.4.0-1.20.jar

📊 Create Kafka Source Table

Open SQL client again:

docker exec -it flink-jobmanager /opt/flink/bin/sql-client.sh

Run:

DROP TABLE IF EXISTS kafka_source;

CREATE TABLE kafka_source ( value STRING ) WITH ( 'connector' = 'kafka', 'topic' = 'iot-topic', 'properties.bootstrap.servers' = 'kafka:29092', 'properties.group.id' = 'flink-group-fixed', 'scan.startup.mode' = 'earliest-offset', 'value.format' = 'raw' );

▶️ Start Streaming Query SELECT * FROM kafka_source;

👉 Now send data using Python or Kafka producer.

📊 PostgreSQL + Grafana 🌐 Grafana Access http://localhost:3000

Default login:

Username: admin Password: admin

🗄️ Access PostgreSQL docker exec -it postgres psql -U postgres -d intrusiondb

🔌 Add JDBC Connector (Flink → PostgreSQL)

Download Connector

docker exec flink-jobmanager wget -P /opt/flink/lib
https://repo.maven.apache.org/maven2/org/apache/flink/flink-connector-jdbc/3.3.0-1.20/flink-connector-jdbc-3.3.0-1.20.jar

docker exec flink-taskmanager wget -P /opt/flink/lib
https://repo.maven.apache.org/maven2/org/apache/flink/flink-connector-jdbc/3.3.0-1.20/flink-connector-jdbc-3.3.0-1.20.jar

Restart Flink docker restart flink-taskmanager docker restart flink-jobmanager

📊 PostgreSQL Table DROP TABLE IF EXISTS attack_summary;

CREATE TABLE attack_summary ( label TEXT PRIMARY KEY, avg_rate DOUBLE PRECISION, event_count BIGINT, ts TIMESTAMP );

🔄 Create Flink Sink Table CREATE TABLE attack_summary_sink ( ts TIMESTAMP(3), label STRING, avg_rate DOUBLE, event_count BIGINT ) WITH ( 'connector' = 'jdbc', 'url' = 'jdbc:postgresql://postgres:5432/intrusiondb', 'table-name' = 'attack_summary', 'username' = 'postgres', 'password' = 'postgres' );

🚀 Insert Streaming Data into PostgreSQL INSERT INTO attack_summary_sink SELECT JSON_VALUE(value, '$.label') AS label, AVG(CAST(JSON_VALUE(value, '$.Rate') AS DOUBLE)) AS avg_rate, COUNT(*) AS event_count, CURRENT_TIMESTAMP AS ts FROM kafka_source GROUP BY JSON_VALUE(value, '$.label');

✅ What This Does Streams IoT data from Kafka Processes using Flink Aggregates attack statistics Stores results in PostgreSQL Visualizes in Grafana
