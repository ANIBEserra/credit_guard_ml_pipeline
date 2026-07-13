<p align="center">
  <b>Languages:</b>
  <a href="README.md">🇧🇷 Português</a> | 
  🇺🇸 English | 
  <a href="README.es.md">🇪🇸 Español</a>
</p>

---

Access Looker Studio to view the monitoring dashboard: [Credit Guard Pipeline](https://datastudio.google.com/s/oUAEZgpbSvY)

# Serverless Pipeline for Ingestion and Data Governance of CNPJ Registration Data (GCP & GitHub Actions)

## 🗺️ Navigation Summary

* [📌 1. Project Purpose & Business Vision](#-1-project-purpose--business-vision)
* [🏗️ 2. Serverless Medallion Architecture](#️-2-serverless-medallion-architecture)
* [📁 3. Repository Organization & Folder Structure](#-3-repository-organization--folder-structure)
* [🛡️ 4. Data Governance & Metadata Management](#️-4-data-governance--metadata-management)
* [📊 5. Pipeline Monitoring & Operational Insights](#-5-pipeline-monitoring--operational-insights)

---

## 📌 1. Project Purpose & Business Vision

In modern corporate environments, consolidating registration data from public external sources (such as the Federal Revenue) usually faces chronic challenges: complex and nested JSON payloads, lack of standardization, absence of clear data dictionaries, and exposure risks of sensitive personal data from individuals (partners).

The objective of this project was to build a **100% serverless and low-cost (FinOps) data pipeline** to centralize, structure, and govern CNPJ and Board of Partners and Administrators (QSA) data consumed via *BrasilAPI*. 

The project focuses on excellence in **Data Engineering and Governance**, demonstrating:
* **Efficient Automation:** Scheduled ingestion via CI/CD without the need to maintain dedicated servers running 24/7.
* **Scalable Architecture:** Optimized storage using the `.parquet` columnar format to reduce analytical scan costs.
* **Governance Culture:** Practical application of data catalog, variable dictionary, and privacy techniques (*Privacy by Design*) directly at the delivery layer.

## 🏗️ 2. Serverless Medallion Architecture

The project architecture was designed following the principles of decoupled layers (Medallion) and managed cloud computing. The workflow ensures efficient asynchronous processing of large data volumes, focusing on storage and query cost optimization.

```text
  [ BrasilAPI ]
        │ (Asynchronous Request / Python)
        ▼
 ┌────────────────────────────────────────┐
 │             GitHub Actions             │ ◄── [ Cron Trigger / CI/CD ]
 └────────────────────────────────────────┘
        │
        ├─► [ 1. JSON Extraction ] ──► Google Cloud Storage (Raw Bucket / Bronze)
        │
        └─► [ 2. Conversion &    ] ──► Google Cloud Storage (Structured Bucket / Silver)
              .parquet Processing
        │
        ▼
 ┌────────────────────────────────────────┐
 │         Google BigQuery (OLAP)         │ ◄── [ Silver Tables Storage ]
 └────────────────────────────────────────┘
        │
        ▼
 ┌────────────────────────────────────────┐
 │           Looker Studio (BI)           │ ◄── [ Data Delivery & Masking ]
 └────────────────────────────────────────┘
```

 ### 🧱 Layer Technical Details

* **Bronze Layer (Raw/JSON):** Data returned from the API in raw JSON format is saved directly to Google Cloud Storage (GCS). This preserves the immutability of the original data and allows for future reprocessing (replayability) without burdening the source API.
* **Processing and Optimization (.parquet):** Using Python, the JSON payload is parsed, cleaned, and converted into the `.parquet` columnar format. This step drastically reduces the final file size in the Data Lake and optimizes read speeds for subsequent analytical stages.
* **Silver Layer (Structured/BigQuery):** The structured `.parquet` files in GCS are integrated into Google BigQuery. In this analytical layer, data receives rigid typing, well-defined schemas, and is organized relationally (Company Table and Board of Partners Table).
* **Delivery Layer (BI/Looker Studio):** BigQuery exposes the tables directly to Looker Studio, where operational and volumetric metrics are displayed seamlessly, eliminating the need for complex runtime processing on the dashboard.

## 📁 3. Repository Organization & Folder Structure

The project was structured in a practical and organized manner, separating execution scripts, metadata configurations, and supporting documentations.

├── configs/
│   ├── __init__.py
│   └── mapping.py           # Mapping dictionaries (From/To) and column renaming
├── data/
│   └── input/
│       └── cnpjs.csv        # Initial list of CNPJs serving as input for the API search
│   ├── raw/                 # [Dynamic] Created at runtime to temporarily store raw JSON files
│   └── silver/              # [Dynamic] Created at runtime to temporarily store .parquet files
├── docs/                    # Auxiliary documentations and project data dictionaries
├── notebooks/
│   ├── analyse_silver.ipynb # Analysis and validation of structured data in the Silver layer
│   └── brasil-api.ipynb     # Jupyter Notebook used for initial exploration of BrasilAPI
├── scripts/
│   └── main.py              # Main script containing extraction, cleaning, and loading functions for GCP
├── .gitignore               # Definition of local files and credentials that should not be versioned
├── README.md                # Complete documentation of the ecosystem
└── requirements.txt        # Project dependencies (pandas, pyarrow, google-cloud-storage, etc.)

## 🛡️ 4. Data Governance & Metadata Management

The governance of this pipeline was designed to ensure traceability and control over the data lifecycle.

### ⏳ 1. Data Lifecycle Management
The pipeline adopts a persistence policy to ensure compliance with LGPD (Brazilian General Data Protection Law) and good data governance practices:
* **Ingestion and Local Disposal:** Files generated in the `data/raw/` and `data/silver/` folders exist only during script execution on the runner and are purged immediately after cloud loading.
* **Layered Persistence (GCP):** Raw data remains immutable in Cloud Storage (Bronze) for auditing purposes, while the analytical layer (Silver) in BigQuery is updated for consumption, ensuring a clean and auditable history.

### 📋 2. Compliance Matrix and Data Classification

| Data Domain | Main Attributes | Purpose of Use | Legal Basis (LGPD) | Security Classification |
| :--- | :--- | :--- | :--- | :--- |
| **Corporate Registration (PJ)** | `NRCNPJ`, `NMRAZSOC`, `NMFANT`, `VLCPTSOC`, `IDMTZFIL`, `DSIDMTZFIL` | Centralization, hygiene, and data enrichment of legal entities for operational risk mitigation, fraud prevention (KYC), and data validation across business ecosystems. | **Legitimate Interest**<br>(Art. 7, IX)<br><br>**Compliance with Legal or Regulatory Obligation**<br>(Art. 7, II - Compliance with AML/Regulations) | **Public**<br>(Institutional data from public records at the Federal Revenue) |
| **QSA** *(Board of Partners)* | Partner Name, CPF (masked/partial), Partner Qualification | Evaluation of corporate relationships, identification of ultimate beneficial owners, and generation of analytical monitoring dashboards in Looker Studio. | **Legitimate Interest**<br>(Art. 7, IX - Business protection)<br><br>**Compliance with Legal or Regulatory Obligation**<br>(Art. 7, II - Compliance and Governance standards) | **Personal / PII**<br>(Data belonging to individuals. Requires dynamic masking at the visualization layer and ephemeral control with local disposal via `clean_local_temp_files`). |

### 📖 3. Data Catalog & Lineage
The data structure is mapped in a declarative and centralized manner:
* **Schema Centralization:** BigQuery type, key, and relationship definitions are controlled via code, ensuring that the ingested volume strictly respects the established data contract.
* **Context Separation:** Clear division between the Company's registration data entity and the Board of Partners and Administrators (QSA) entity.

### 📋 4. Log Table and Monitoring
To guarantee pipeline observability, the script populates a dedicated operational log table in BigQuery at every execution. This table records:
* **Ingestion Metadata:** Processing date and time, volume of CNPJs successfully queried, number of API failures, and final load status (Success/Error). This allows tracking the health of the analytical flow without needing to open infrastructure consoles.

### 🔤 5. Nomenclature Standardization (DATASUS Pattern)
The column dictionary within the `configs/mapping.py` file adopts a vowel compression and taxonomic prefix methodology inspired by DATASUS:

The from/to explicitly maps types and short aliases:
* `cnpj` ➔ `NRCNPJ` (CNPJ Number)
* `razao_social` ➔ `NMRAZSOC` (Corporate Name)
* `nome_fantasia` ➔ `NMFANT` (Trade Name)
* `capital_social` ➔ `VLCPTSOC` (Value of Capital Stock)
* `identificador_matriz_filial` ➔ `IDMTZFIL` (Identifier of Main/Branch)
* `descricao_identificador_matriz_filial` ➔ `DSIDMTZFIL` (Description of Identifier Main/Branch)

This approach drastically reduces metadata noise, saves column storage bytes, standardizes Data Warehouse logical keys, and ensures that relational modeling remains clean and predictable for any analyst in the organization.

## 📊 5. Pipeline Monitoring & Operational Insights

The final stage of the ecosystem consists of exposing the processed and structured data from the Silver layer (BigQuery) to **Looker Studio**, where the ingestion flow and enterprise data volume are visually monitored.

### 📈 Monitoring Metrics & Observability
The technical dashboard consumes the log table generated by the pipeline directly, allowing the tracking of vital DataOps indicators:
* **Ingestion Success Rate:** Percentage of requests sent to BrasilAPI that returned successfully (*Status 200*) vs. registration failures (invalid or non-existent CNPJs).
* **Daily Volumetrics:** Monitoring the volume of rows injected per execution to ensure there are no drastic drops or anomalies in the processing flow.
* **FinOps Metrics:** Traceability of execution time and volume of data transferred, helping ensure that the pipeline operates within the GCP *Free Tier* limits.

### 💡 Extraction of Basic Registration Insights
Beyond pipeline monitoring, the dashboard consolidates general views over the processed company portfolio to provide business intelligence:
* **Geographical Distribution:** Concentration of active companies across the national territory by State (UF) and municipality (using the mapped `cd_municipio` field).
* **Size and Capital Analysis:** Grouping the financial volume of organizations based on the distribution of the `VLCPTSOC` (Capital Stock) field.
* **Corporate Structure Vision:** Quantitative mapping of the number of partners per organization, allowing an understanding of the density of corporate links in the queried ecosystem.