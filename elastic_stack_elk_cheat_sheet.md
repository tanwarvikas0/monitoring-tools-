# Elastic Stack (ELK) — Quick Notes

A concise overview of the core Elastic Stack components, extensions, and data shippers.

---

## 1. Core ELK Stack

### Elasticsearch
* **What it is:** Open-source, distributed full-text search and analytics engine.
* **Tech Base:** Built in Java; highly scalable and easy to use.
* **Key Capabilities:**
  * Application search integration
  * Autocomplete / typeahead
  * Typo correction (fuzzy search)
  * Synonym handling

### Kibana
* **What it is:** Web-based UI and dashboard for the Elastic Stack.
* **Primary Role:** Data exploration, analysis, and interactive visualization (charts, graphs, maps).

### Logstash
* **What it is:** Free, open server-side data processing pipeline.
* **Primary Role:** Ingests raw data from diverse sources, parses and transforms it, and outputs to storage destinations ("stashes") like Elasticsearch or Kafka.
* **Pipeline Flow:**
  $$\text{Data Source (Logs)} \longrightarrow \text{Logstash (Ingest \& Transform)} \longrightarrow \text{Elasticsearch / Kafka}$$

---

## 2. X-Pack (Extensions & Security)

Adds enterprise and operational capabilities to Elasticsearch and Kibana:

* **Security:** User authentication (LDAP/AD/tokens) and role-based access control (RBAC).
* **Monitoring & Alerting:** Tracks cluster health and metrics; notifies on thresholds.
* **Machine Learning:** Automated anomaly detection and pattern analysis directly inside the stack.
* **Graph:** Uncovers relevance and relationships in data for recommendation engines (accessible via Graph API).
* **Elasticsearch SQL:**
  * **SQL API:** Run direct SQL queries against Elasticsearch indexes.
  * **Translate API:** Translates SQL queries into native Elasticsearch **Query DSL**.

---

## 3. Beats (Data Shippers)

Lightweight, single-purpose agents installed directly on edge machines and servers to capture and stream telemetry to Logstash or Elasticsearch.

### The Beats Family Overview

| Beat | Type | Target Data |
| :--- | :--- | :--- |
| **Filebeat** | Logs | System, server, and application log files |
| **Metricbeat** | Metrics | System and service CPU, memory, and disk stats |
| **Packetbeat** | Network | Real-time network protocols and traffic analysis |
| **Winlogbeat** | Windows Events | Windows system, application, and security event logs |
| **Auditbeat** | Security Audit | Linux audit framework events and file integrity monitoring (FIM) |
| **Heartbeat** | Uptime | Active probing (ICMP, TCP, HTTP) for service availability |
| **Functionbeat** | Serverless | Cloud-native logs and serverless streams (e.g., AWS Lambda) |

---

## Architecture Flow

$$\text{Data Sources} \xrightarrow{\text{Beats}} \begin{bmatrix} \text{Logstash} \\ \text{(Transform/Parse)} \end{bmatrix} \xrightarrow{\text{Index}} \begin{bmatrix} \text{Elasticsearch} \\ \text{(Store/Search/ML)} \end{bmatrix} \xrightarrow{\text{Visualize}} \begin{bmatrix} \text{Kibana} \\ \text{(Dashboards)} \end{bmatrix}$$