<p align="center">
  <b>Languages:</b>
  <a href="README.md">🇧🇷 Português</a> | 
  🇺🇸 English | 
  <a href="README.es.md">🇪🇸 Español</a>
</p>

---

Access Looker Studio to view the monitoring portal: [Credit Guard Pipeline](https://datastudio.google.com/s/oUAEZgpbSvY)


# Case Study: CreditGuard — Governed Data Engineering Pipeline

This project presents the complete lifecycle of a governed and automated data pipeline for a financial risk analysis company, following real market practices, information security, and compliance with LGPD (Brazilian General Data Protection Law).

---

## 🗺️ Navigation Summary

* [1. Business Context, Challenge, and Purpose](#1-business-context-challenge-and-purpose)
* [2. Project Requirements](#2-project-requirements)
* [3. Serverless Medallion Architecture](#3-serverless-medallion-architecture)
* [4. Repository Organization & Folder Structure](#4-repository-organization--folder-structure)
* [5. Data Governance Policy & Metadata Management](#5-data-governance-policy--metadata-management)
* [6. Testing and Delivery Cycle](#6-testing-and-delivery-cycle)
* [7. Pipeline Monitoring & Operational Insights](#7-pipeline-monitoring--operational-insights)
* [8. Project Deliverables](#8-project-deliverables)

---

## 1. Business Context, Challenge, and Purpose

**CreditGuard** is a credit intelligence fintech specialized in risk assessment and compliance (*Know Your Customer* - KYC) for small and medium-sized B2B commercial transactions.

In the credit ecosystem, speed and the cost of information acquisition are inherent factors. To grant financial limits, the fintech consumes data from **paid credit bureaus** (such as Serasa). However, making paid API calls for absolutely every received proposal generates an operational cost (e.g., Customer Acquisition Cost - CAC).

### Business Pain
Before implementing this project, CreditGuard suffered from three major pain points in its credit approval pipeline:
* **Costs and Inefficiency:** The fintech spent excessive resources querying paid bureaus for companies that could already be summarily rejected because they were inactive at the Brazilian Federal Revenue or had CNAEs outside the credit policy.
* **Systemic and Operational Bottleneck:** Occasional queries to public sources were performed in a fragmented and manual way by the risk team, preventing credit approval from being carried out 100% automatically by a Rules Engine.
* **Lack of Auditability and QSA Traceability:** The corporate structure was not stored in a relational manner. This prevented complex risk analyses (e.g., automatically identifying whether a partner of the current company had already bankrupted a previous company in the fintech's base) and made it impossible to audit the exact situation of the company on the day the credit was granted.

### The Solution & Objective
To solve this pain, a **100% serverless and governed data pipeline** was developed, acting as a **primary data enrichment layer (Tier 1)** using free public sources (BrasilAPI).

This pipeline intercepts and processes CNPJ and Corporate Structure (QSA) data before calls to paid bureaus become necessary, focusing on:
* **FinOps Strategy:** Automated and structured ingestion allows the fintech's decision engine to deny unfeasible proposals at zero cost, using public data, triggering paid APIs only for customers who pass the first filter.
* **Engineering for Scale Decision-Making:** Storage optimized in the columnar `.parquet` format and consolidation in BigQuery (Data Warehouse), allowing Machine Learning models and risk dashboards to consume thousands of records instantly.
* **Governance Culture and LGPD:** Practical application of *Privacy by Design*. Personal data of partners (PII) is masked in the delivery layer, ensuring full compliance with data protection laws while maintaining a standardized variable dictionary.

---

## 2. Project Requirements

### Functional Requirements (What the pipeline delivers)
* **FR-01:** Automated batch extraction of registration data for CNPJs provided as input.
* **FR-02:** Secure storage of the raw payload in the cloud to enable future reprocessing.
* **FR-03:** Taxonomic standardization and cleaning of company and partner data, removing special characters and applying strict typing.
* **FR-04:** Creation of a continuous monitoring matrix to evaluate the fill integrity and completeness of loaded columns.
* **FR-05:** Provision of a central log table for auditing pipeline executions (success, processing time, errors, and volumes).

### Non-Functional Requirements (Architectural and technical rules)
* **NFR-01 (FinOps/Cost):** The project must operate 100% within the free tier limits of cloud services (GCP) and orchestration runners.
* **NFR-02 (Efficient Storage):** Use of the columnar `.parquet` format to reduce transport costs in the Data Lake and increase read performance.
* **NFR-03 (Security):** No raw personal data (PII) of individuals should remain saved on temporary servers or local runners after the execution ends.
* **NFR-04 (Automation):** Scheduling and isolated execution via CI/CD pipeline, eliminating dependency on active local infrastructures.

---

## 3. Serverless Medallion Architecture

The project architecture was designed following the principles of layer decoupling (Medallion) and managed cloud computing. The flow ensures efficient data processing asynchronously, focusing on optimizing storage and query costs.

```text
  [ BrasilAPI ]
        │ (Asynchronous Request / Python)
        ▼
 ┌────────────────────────────────────────┐
 │            GitHub Actions              │ ◄── [ Cron Trigger / CI/CD ]
 └────────────────────────────────────────┘
        │
        ├─► [ 1. JSON Extraction ] ──► Google Cloud Storage (Raw / Bronze Bucket)
        │
        └─► [ 2. Transformation &    ] ──► Google Cloud Storage (Structured / Silver Bucket)
             .parquet Processing
        │
        ▼
 ┌────────────────────────────────────────┐
 │        Google BigQuery (OLAP)          │ ◄── [ Silver Table Storage ]
 └────────────────────────────────────────┘
        │
        ▼
 ┌────────────────────────────────────────┐
 │         Looker Studio (BI)             │ ◄── [ Delivery and Data Masking ]
 └────────────────────────────────────────┘