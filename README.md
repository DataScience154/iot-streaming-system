# IoT Intrusion Detection Streaming Pipeline (Real-Time Analytics)

## 📌 Overview
This project implements a real-time IoT Intrusion Detection System (IDS) using stream processing technologies. It simulates IoT network traffic and processes it through a streaming pipeline to detect anomalies and cyberattacks such as DDoS and botnet activity.

The system uses:
- Python (data ingestion)
- Apache Kafka (streaming)
- Apache Flink (real-time processing)
- PostgreSQL (storage)
- Grafana (visualisation)
- Docker (containerisation)

---

## 🧠 Architecture
Pipeline flow:

Python → Kafka → Flink → PostgreSQL → Grafana

- Python streams IoT data as JSON
- Kafka handles message streaming
- Flink processes data in real time
- PostgreSQL stores aggregated results
- Grafana visualises insights

---

## ⚙️ Requirements
- Docker & Docker Compose
- Python 3.10+
- pip

---

## Setup Instructions

1. Clone the Repository


2. Start All Services (Docker)
docker-compose up -d

3. Verify Running Containers
docker ps

4. Create Kafka Topic
docker exec kafka /opt/kafka/bin/kafka-topics.sh --create --topic iot-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1

5. Run Kafka Consumer (Optional - Debug)
docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh --topic iot-topic --bootstrap-server localhost:9092 --from-beginning

**Python Environment Setup**
Method 1: Virtual Environment
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
Method 2: Conda
conda create -n iot-env python=3.10
conda activate iot-env
pip install -r requirements.txt

6. Run Python Producer
python producer.py

**Apache Flink Setup**
Open Flink SQL Client
docker exec -it flink-jobmanager /opt/flink/bin/sql-client.sh

⚠️ Fix Kafka Connector (IMPORTANT)
If Kafka table is not detected:
docker exec flink-jobmanager wget -P /opt/flink/lib https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.4.0-1.20/flink-sql-connector-kafka-3.4.0-1.20.jar

docker exec flink-taskmanager wget -P /opt/flink/lib https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.4.0-1.20/flink-sql-connector-kafka-3.4.0-1.20.jar
Restart Flink:
docker restart flink-taskmanager
docker restart flink-jobmanager
Verify:
docker exec flink-jobmanager ls /opt/flink/lib
docker exec flink-taskmanager ls /opt/flink/lib
Expected:
flink-sql-connector-kafka-3.4.0-1.20.jar

Create Kafka Source Table
DROP TABLE IF EXISTS kafka_source;

CREATE TABLE kafka_source (
    value STRING
) WITH (
    'connector' = 'kafka',
    'topic' = 'iot-topic',
    'properties.bootstrap.servers' = 'kafka:29092',
    'properties.group.id' = 'flink-group-fixed',
    'scan.startup.mode' = 'earliest-offset',
    'value.format' = 'raw'
);

Start Streaming Query
SELECT * FROM kafka_source;

**PostgreSQL Setup**
Access PostgreSQL
docker exec -it postgres psql -U postgres -d intrusiondb

Create Table
DROP TABLE IF EXISTS attack_summary;

CREATE TABLE attack_summary (
    label TEXT PRIMARY KEY,
    avg_rate DOUBLE PRECISION,
    event_count BIGINT,
    ts TIMESTAMP
);

**Flink → PostgreSQL Connector**
Download JDBC Connector
docker exec flink-jobmanager wget -P /opt/flink/lib https://repo.maven.apache.org/maven2/org/apache/flink/flink-connector-jdbc/3.3.0-1.20/flink-connector-jdbc-3.3.0-1.20.jar

docker exec flink-taskmanager wget -P /opt/flink/lib https://repo.maven.apache.org/maven2/org/apache/flink/flink-connector-jdbc/3.3.0-1.20/flink-connector-jdbc-3.3.0-1.20.jar
Restart Flink:
docker restart flink-taskmanager
docker restart flink-jobmanager

Create Sink Table
CREATE TABLE attack_summary_sink (
    ts TIMESTAMP(3),
    label STRING,
    avg_rate DOUBLE,
    event_count BIGINT
) WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:postgresql://postgres:5432/intrusiondb',
    'table-name' = 'attack_summary',
    'username' = 'postgres',
    'password' = 'postgres'
);

Insert Streaming Data
INSERT INTO attack_summary_sink
SELECT 
    JSON_VALUE(value, '$.label') AS label,
    AVG(CAST(JSON_VALUE(value, '$.Rate') AS DOUBLE)) AS avg_rate,
    COUNT(*) AS event_count,
    CURRENT_TIMESTAMP AS ts
FROM kafka_source
GROUP BY JSON_VALUE(value, '$.label');

📈 Grafana Dashboard
Access Grafana
http://localhost:3000
Login:
•	Username: admin
•	Password: admin

✅ Results
The system:
•	Streams IoT data in real time
•	Detects abnormal traffic spikes
•	Aggregates attack statistics
•	Stores results in PostgreSQL
•	Visualises data using Grafana dashboards

