# 🧩 CDPForge – Modular & Extensible Customer Data Platform

**CDPForge** is a proprietary, Kubernetes-native Customer Data Platform designed to be **modular**, **real-time**, and **easily extensible** via plugins. It collects, unifies, governs, and activates customer data — providing identity resolution, audience segmentation, on-site personalization, a native consent manager, and flexible APIs.

---

## ✨ Platform Capabilities

A single workspace to collect, unify, govern, and activate your customer data:

| Capability | Description |
|------------|-------------|
| **Multi-channel collection** | Collect events from websites, apps, and APIs with a lightweight tracker that queues events and never loses data. |
| **Identity resolution** | Merge anonymous and identified sessions into a single unified profile (Person) via device IDs and external identifiers. |
| **Audience segmentation** | Build segments with a visual query builder and preview the matching profiles before activation. |
| **Real-time enrichment** | A plugin pipeline enriches every event in real time — geo, identity, behavioral scoring — over an Apache Pulsar bus. |
| **Event-driven triggers** | Automate actions that react to events or segment membership, with full visibility on every execution. |
| **On-site personalization** | Serve targeted HTML components to qualified visitors, with activation rules based on URL and DOM. |
| **Consent management** | A native consent manager and preference center: a per-client purpose catalog, a consent ledger, and ingestion gating for required purposes. |
| **Governance & GDPR** | Audit logs of every action and tooling to handle GDPR deletion and data export requests. |
| **AI assistant** | Query your data in natural language and get guided through operations, with a configurable LLM provider per client. |
| **Flexible APIs & SDK** | REST APIs to manage clients, instances, segments, and profiles, plus an SDK to build custom pipeline plugins. |
| **Privacy Sandbox ready** | Support for the Google Topics API (`browsingTopics`) to capture contextual interests in a privacy-friendly way. |
| **Kubernetes-native** | A modular, scalable architecture where each module is deployable and scalable independently. |

---

## 🏗️ Architecture Overview

### Core Modules

| Module            | Description |
|-------------------|-------------|
| **Input Manager**   | Ingests data via API or file uploads. Validates and forwards events onto the message bus. |
| **Pipeline (ETL)**  | Multi-stage, real-time ETL pipeline with plugin support (input, processing, output). |
| **Output API**      | Provides access to all data and platform configuration via REST APIs. |
| **Identity Resolution** | Unifies anonymous and identified sessions into a single Person via device IDs and external identifiers. |
| **Consent Manager** | Per-client purpose catalog, consent ledger, preference center, and ingestion gating for required purposes. |
| **Auth System**     | Internal JWT-based access control. |
| **Plugin Manager**  | Handles plugin/module installation, registration, and configuration. |

---

## 🔁 Data Processing Pipeline

Each stage of the ETL pipeline allows for:

- **Blocking Plugins**: must complete before processing continues (they modify data).
- **Non-blocking Plugins**: run in parallel; observe data but do not modify it.

Events flow over an **Apache Pulsar** bus, so each module can be deployed and scaled independently.

---

## 🔌 Plugin System

### Types of Plugins

| Type               | Description |
|--------------------|-------------|
| **Input Plugins**    | Ingest from various data sources (web tracker, file imports, external systems). |
| **Pipeline Blocking Plugins** | Real-time enrichment or data modification within the pipeline (e.g. geo, identity, scoring). |
| **Pipeline Non-blocking Plugins** | Asynchronous observers for export, logging, or analytics. |
| **Output Plugins**   | Forward processed data to external destinations and activation channels. |

All plugins:
- Are deployed as independent services/pods.
- Communicate via the message bus (Apache Pulsar).
- Are built with the official **Plugin SDK** and managed by the **Plugin Manager**.

---

## 🗃️ Database Architecture

| Database         | Purpose |
|------------------|---------|
| **Elasticsearch** | Event storage and querying. |
| **MySQL**         | Configs, plugin metadata, profiles, consent, and admin data. |

---

## 🚀 Kubernetes Deployment

- Fully deployable on **Kubernetes** using a **Helm Chart**.
- Plugin activation and configuration via `values.yaml`.
- Secure, modular, and scalable microservice deployment.

---

## 🔐 Security & Privacy

- JWT-based authentication across modules.
- All traffic scoped to the internal K8s network.
- Plugins only interact via the message bus — reducing attack surface.
- Native consent management with ingestion gating, plus audit logs and GDPR deletion/export tooling.

---

## ⚙️ Technologies

| Area             | Technology |
|------------------|------------|
| Core Backend     | Node.js |
| Message Bus      | Apache Pulsar |
| Databases        | Elasticsearch, MySQL |
| API Layer        | REST |
| Auth             | JWT (OAuth2-ready) |
| Deployment       | Kubernetes + Helm |

---

## 🤝 Contributing

CDPForge is proprietary software. Contributions are accepted by arrangement: any pull request requires accepting the terms in our [Contributor License Agreement (CLA)](PROFILE/CLA.md) — see the [Contributing Guidelines](PROFILE/CONTRIBUTING.md) for details.

---

## 📄 License

This project is proprietary software. All rights reserved. Use is governed by the [CDPForge Proprietary License](PROFILE/LICENSE).
