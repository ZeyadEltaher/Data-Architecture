# Data Architecture

## 1. Overview

This project is a multi-pipeline data system designed to handle:

1. Real-time sensor data from sports fields for live analytics (App 1).
2. Global application requests for player stats, analytics, and scouting (App 2).
3. Historical data storage for analytics, reporting, and business intelligence (Data Warehouse).

The system integrates on-premise servers at each stadium and cloud-hosted services for global access and storage.

---

## 2. Pipelines

### 2.1 Real-time Pipeline (On-Premise)

**Purpose:** Handle sensor data in real-time and provide live analytics for coaches and medical staff.

**Flow:**
Sensors → MQTT → Kafka Connect (Source) → Kafka → PySpark → Kafka Connect (Sink) → InfluxDB → Grafana / Application 1

**Details:**

- **Sensors:** Collect player metrics (heart rate, speed, position, etc.).
- **MQTT:** Lightweight protocol to transmit sensor data to the local server.
- **Kafka + Kafka Connect (Source):** Receive messages from MQTT and distribute them to streaming processors.
- **PySpark (Structured Streaming):** Process data in real-time (cleaning, aggregations, metrics calculations).
- **Kafka Connect (Sink):** Send processed data to InfluxDB.
- **InfluxDB:** Time-series database for fast, temporary storage of sensor metrics.
- **Grafana / Application 1:** Visualize data live for coaches on-site (iPads or tablets).

**Notes:**

- **Deployment:** All on-premise servers (one per stadium).
- **Security:** Each stadium network isolated; only authorized devices can access data.
- **Latency:** Sub-second latency required for live analytics.

---

### 2.2 Database Storage Pipeline (Cloud)

**Purpose:** Handle frequent requests from App 2 users globally. Provides persistent storage for live metrics.

**Flow:**
Airflow Job (every 2 minutes):
1. Read from last checkpoint in InfluxDB.
2. Load into Google Cloud SQL (PostgreSQL).

**Details:**

- **Google Cloud SQL (PostgreSQL):** Handles concurrent reads/writes from users worldwide.
- **Airflow:** Orchestrates ETL tasks every 2 minutes to synchronize data from InfluxDB → Cloud SQL (PostgreSQL).
- **App 2:** Fetches data from Cloud SQL, enabling global users (coaches, scouts) to access player analytics.

**Notes:**

The same PostgreSQL database serves two purposes:

1. Immediate application requests (App 2).
2. Feeding permanent storage pipeline (ETL to BigQuery).

Data is partitioned via checkpoints to avoid duplicates.

---

### 2.3 Permanent Storage Pipeline (Cloud)

**Purpose:** Maintain historical data and support long-term analytics and reporting.

**Flow:**
Airflow Job (daily):
1. Read from last checkpoint in Google Cloud SQL for PostgreSQL.
2. Load into Google BigQuery.

**Details:**

- **Airflow on Google Cloud:**
  - Schedules daily ETL jobs to move data from Cloud SQL for PostgreSQL → BigQuery.
  - Tracks checkpoints to ensure incremental loading.

- **Google BigQuery:**
  - Cloud data warehouse for historical data storage.
  - Optimized for large-scale analytics and BI reporting.
  - Enables dashboards, trends, and historical comparisons.

---

## 3. Data Flow Summary

**Real-time Pipeline - On-Premise**
- Sensors → MQTT → Kafka → PySpark → InfluxDB → Grafana / Application 1 → Coaches & Scouts

**Database Storage Pipeline - Cloud**
- Airflow Job (InfluxDB → Cloud SQL for Postgres) → Application 2 → Global users

**Permanent Storage Pipeline - Cloud**
- Airflow Job (Cloud SQL → BigQuery) → Power BI (Historical Analytics / Reports)

---

## 4. Tools and Services

### 4.1 On-Premise (Stadium Servers)

| Tool | Purpose |
|------|----------|
| MQTT | Lightweight messaging protocol for sensor data. |
| Kafka + Kafka Connect | Message broker to handle streaming data and distribution. |
| PySpark (Structured Streaming) | Real-time processing and aggregation. |
| InfluxDB | Time-series database for temporary live metrics. |
| Grafana / Application 1 | Live visualization for on-site coaches. |
| Airflow (local) | Orchestrating real-time ETL at the stadium. |

### 4.2 Cloud-Hosted

| Tool | Purpose |
|------|----------|
| Airflow (Cloud) | Orchestration of ETL pipelines (daily & scheduled). |
| Google Cloud SQL (PostgreSQL) | Persistent relational database for App 2 global requests. |
| Google BigQuery | Data warehouse for historical analytics and reporting. |

---

## 5. Deployment Architecture

### 5.1 On-Premise

- Each stadium has one local server:
  - Runs Docker containers for Kafka, PySpark, InfluxDB, Grafana.
  - Receives sensor data via MQTT.
  - Handles real-time analytics with low latency (<1s).
  - Network isolated → only authorized devices can access Grafana / Application 1.

### 5.2 Cloud

- **Cloud SQL for PostgreSQL:** Single instance with optional read replicas for global App 2 users.
- **Airflow:** Schedules ETL jobs for:
  1. InfluxDB → Cloud SQL for PostgreSQL (every 2 minutes).
  2. Cloud SQL for PostgreSQL → BigQuery (daily).
- **BigQuery:** Historical analytics & reports accessible via BI tools or dashboards.

> This architecture ensures low-latency live analytics, global availability, and scalable historical analytics for the sports data system.
