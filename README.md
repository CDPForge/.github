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
| **Auth System**     | Internal JWT-based access control. |
| **Plugin Manager**  | Handles plugin/modules installation, registration, and configuration. |

---

## 🔁 Data Processing Pipeline

Each stage of the ETL pipeline allows for:

- **Blocking Plugins**: must be executed before proceeding (they modify data).
- **Non-blocking Plugins**: run in parallel; observe data but do not modify it.

---

## 🔌 Plugin System

### Types of Plugins

| Type               | Description |
|--------------------|-------------|
| **Input Plugins**    | Ingest from various data sources (CSV, Salesforce, etc.). |
| **Pipeline Blocking Plugins** | Real-time enrichment or data modification within the pipeline. |
| **Pipeline Non-blocking Plugins** | Asynchronous observers for export, logging, or analytics. |
| **Batch Plugins**    | Periodic data jobs (e.g. audience builders, transformations, sync with external services). |

All plugins:
- Are deployed as independent services/pods.
- Communicate via Kafka (or other event bus).
- Are managed by the **Plugin Manager**.

---

## 🗃️ Database Architecture

| Database         | Purpose |
|------------------|---------|
| **Elasticsearch** | Event storage and querying. |
| TBD: **GraphDB (Neo4j, etc.)**   | Identity resolution (ID graph). |
| **MySQL / PostgreSQL**      | Configs, plugin metadata, admin data. |
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
| Message Bus      | Kafka   |
| Databases        | Elasticsearch, MySQL, GraphDB(TBD) |
| API Layer        | REST or GraphQL |
| Auth             | JWT (OAuth2-ready) |
| Deployment       | Kubernetes + Helm |

---

## 🤝 Contributing

We welcome contributions!  
Before submitting a pull request, please read our [Contributing Guidelines](CONTRIBUTING.md) and accept the terms in our [Contributor License Agreement (CLA)](CLA.md).

---

## 📄 License

This project is licensed under the terms of the [GNU General Public License v3.0](LICENSE).
