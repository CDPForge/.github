# 🧩 CDPForge – Modular & Extensible Customer Data Platform

**CDPForge** is an open source, Kubernetes-native Customer Data Platform designed to be **modular**, **real-time**, and **easily extensible** via plugins. It collects, processes, enriches, and stores customer-related data, providing flexible APIs and identity resolution features.

---

## 🏗️ Architecture Overview

### Core Modules

| Module            | Description |
|-------------------|-------------|
| **Input Manager**   | Ingests data via API or file uploads. Validates and forwards to Kafka. |
| **Event Processor** | Multi-stage real-time ETL pipeline with plugin support. |
| **Output API**      | Provides access to all data via REST or GraphQL APIs. |
| **ID Graph Engine** | Resolves user identity via cookies, emails, user IDs using a GraphDB. |
| **DB Proxy Layer**  | Abstracts data access to support interchangeable DB backends. |
| **Auth System**     | Internal JWT-based access control. |
| **Plugin Manager**  | Handles plugin/modules installation, registration, and configuration. |

---

## 🔁 Data Processing Pipeline

Each stage of the ETL pipeline allows for:

- **Blocking Plugins**: must be executed before proceeding (they modify data).
- **Non-blocking Plugins**: run in parallel; observe data but do not modify it.

### Blocking Plugins
- Triggered via Kafka topics (e.g. `stageN.plugins.request`).
- Must respond within a timeout.
- Core merges plugin responses via reducer logic.

### Non-blocking Plugins
- Listen to broadcast topics (e.g. `stageN.broadcast`).
- Do not affect the data flow.
- Used for logging, alerts, external integrations, etc.

---

## 🔌 Plugin System

### Types of Plugins

| Type               | Description |
|--------------------|-------------|
| **Input Plugins**    | Ingest from various data sources (CSV, Salesforce, etc.). |
| **Blocking Plugins** | Real-time enrichment or data modification within the pipeline. |
| **Non-blocking Plugins** | Asynchronous observers for export, logging, or analytics. |
| **Batch Plugins**    | Periodic data jobs (e.g. audience builders, transformations, sync with external services). |

All plugins:
- Are deployed as independent services/pods.
- Communicate via Kafka (or other event bus).
- Are managed by the **Plugin Manager**.

---

## 🗃️ Database Architecture

| Database         | Purpose |
|------------------|---------|
| **MongoDB / Elasticsearch** | Event storage and querying. |
| **GraphDB (Neo4j, etc.)**   | Identity resolution (ID graph). |
| **MySQL / PostgreSQL**      | Configs, plugin metadata, admin data. |
| **DB Proxy**                | Secure, pluggable DB access layer. |

---

## 🚀 Kubernetes Deployment

- Fully deployable on **Kubernetes** using a **Helm Chart** or a custom **Operator**.
- Plugin activation and configuration via `values.yaml`.
- Secure, modular, and scalable microservice deployment.

---

## 🔐 Security Model

- JWT-based authentication across modules.
- All traffic scoped to internal K8s network.
- Plugins only interact via message bus — reducing attack surface.

---

## ⚙️ Technologies

| Area             | Technology |
|------------------|------------|
| Core Backend     | Node.js |
| Message Bus      | Kafka / NATS / Redis Streams |
| Databases        | MongoDB, MySQL, Neo4j |
| API Layer        | REST or GraphQL |
| Auth             | JWT (OAuth2-ready) |
| Deployment       | Kubernetes + Helm |
| Observability    | Prometheus, Grafana, Loki or ELK stack |

---

## 🧠 Example Event Flow

1. Input API receives user event or file.
2. It validates and sends to Kafka topic: `pipeline.stage1.input`.
3. Stage 1 Processor:
    - Sends to blocking plugins → merges results.
    - Broadcasts event to non-blocking plugins.
4. Proceeds to Stage 2, Stage 3, etc.
5. Stores final enriched event in databases.
6. Output APIs serve processed data on request.
