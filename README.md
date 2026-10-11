# AZURE-DATA-FACTORY_project
An end-to-end **hybrid ETL / ELT pipeline built on Azure Data Factory (ADF)** that ingests airline booking data from multiple sources (**on-premises files via a Self-hosted Integration Runtime**, a REST/Web API on GitHub, and a SQL database), transforms it with **Mapping Data Flows** using the **Medallion Architecture (Bronze → Silver → Gold)**, and produces an analytics-ready dataset of top-performing airlines. The whole run is orchestrated by a parent pipeline that sends an **email alert via Azure Logic Apps** when it completes.

## Tech Stack

| Area | Tools |
|---|---|
| Orchestration & ETL | Azure Data Factory (V2), Mapping Data Flows, **Self-hosted Integration Runtime**, Azure Integration Runtime |
| Storage | Azure Storage Accounts (Data Lake / Blob) |
| Database | Azure SQL Database |
| Alerting | Azure Logic Apps (Standard), Outlook connector, Application Insights |
| Version control & DevOps | Git integration (GitHub, `main` branch), Azure DevOps |
| Languages | SQL, ADF expression language |

## Architecture

# Azure Data Engineering Project Architecture

<img width="1774" height="887" alt="image" src="https://github.com/user-attachments/assets/c93c895d-6524-4934-8c6d-d3657fbd2c0c" />




## Pipelines (6)

| Pipeline | Purpose | Key activities |
|---|---|---|
| `onpremingestion` | Ingests multiple on-premises files through the **Self-hosted Integration Runtime** in a single parameterized, reusable run | **ForEach** over a `files` array parameter (e.g. `DimAirline.csv`) → **Copy data**; per-dataset schema mappings passed as parameters (`p_mapping_flight`, `p_mapping_passenger`, `p_mapping_Airlines`) |
| `pipelineApi` | Pulls files from a GitHub-hosted source over HTTP | **Web** activity → **Copy data** (`copyfromGit`) |
| `pipelineSQL` | Incremental load from SQL using a last-load / latest-load comparison | Two **Lookup** activities (`lastload`, `latestload`) → **Copy data** (`Copy data_sql`) → **Copy data** (`replacedata`) |
| `Silverlayer` | Cleans and conforms raw data | **Data Flow** activity running `Datatransformation` |
| `gold_layer` | Builds the business-level output | **Data Flow** activity running `dataseving` |
| `parentpipeline` | End-to-end orchestration | 3 × **Execute Pipeline** (on-prem → API → SQL) → **Web** activity (`alertmessage`) that triggers the email alert |

## Data Flows (2)

### `Datatransformation` (Silver layer)
Five parallel streams, each with source → transformation → **Alter Row** → sink:
- **DimAirline**: Derived Column (`airline_id`, `airline_name`, `country`)
- **DimFlight**: Select (column renaming)
- **DimPassenger**: three chained Derived Columns (`passenger_id`, `full_name`, `gender`, `age`, ...)
- **SQL source**: Cast (data type conversion)
- **DimAirport**: Derived Column (`airport_id`, `airport_name`, `city`, ...)

### `dataseving` (Gold layer)
`FactBookingSQL` **Left Outer Join** `DimAirline` → **Select** → **Aggregate** by `airline_name` → **Window** → **Filter** (top 5) → **Alter Row** → **Sink (gold)**

The result is a gold dataset of the top 5 airlines, ready for reporting.

## Azure Resources

Everything lives in one resource group (`project_resource`):
- Azure Data Factory (V2)
- Storage accounts for raw and curated data
- Azure SQL Server and SQL Database
- Logic App (Standard), Application Insights and Outlook API connections for alerting

- **Self-hosted Integration Runtime** installed on a local machine to securely connect ADF to the on-premises source (hybrid data integration).
- Data flows run on `AutoResolveIntegrationRuntime` (Small compute).

## Repository Structure

```
├── dataflow/             # Mapping Data Flows (Datatransformation, dataseving)
├── dataset/              # 14 datasets (sources and sinks)
├── factory/              # Factory-level configuration
├── integrationRuntime/   # Integration Runtime definitions
├── linkedService/        # Connections to storage, SQL and APIs
├── pipeline/             # 6 pipelines
├── publish_config.json
└── README.md
```

## Key Concepts Demonstrated

- Parameterized, metadata-driven ingestion (ForEach + parameters)
- Hybrid data integration: on-premises to cloud using a Self-hosted Integration Runtime
- Multi-source ingestion: on-prem files, REST/Web API, SQL
- Incremental loading with Lookup activities
- Medallion Architecture (Bronze / Silver / Gold)
- Mapping Data Flows: derived column, select, cast, join, aggregate, window, filter, alter row
- Pipeline orchestration with Execute Pipeline
- Automated alerting using Web activity + Azure Logic Apps
- Git integration (GitHub) and Azure DevOps for version control and DevOps workflow in ADF

## How to Run

1. Create an Azure Data Factory and link it to this repo (Manage → Git configuration → `main` branch).
2. Install and register a **Self-hosted Integration Runtime** on the machine that holds the on-premises files (Manage → Integration runtimes → New → Self-hosted).
3. Create the storage account(s) and Azure SQL Database, then update the **linked services** with your own connection details (store secrets in **Azure Key Vault**; never commit them).
4. Place the source files in the on-prem folder and update the dataset paths.
5. Open `parentpipeline`, click **Debug** (or **Add trigger**), and monitor under **Monitor → Pipeline runs**.
6. Check the gold sink for the top 5 airlines output.

## Screenshots
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/c45641bf-0ad6-486b-b5bf-eb174519a743" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/80ac7252-4198-4a55-8aaa-aa4d8a2a65f4" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/6aa3e91a-3879-4ba0-8d7f-f59f4f9b7444" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2f0eb974-0c44-4340-b168-e0e30002b961" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/1401661b-2900-4783-a76e-e7de30dff197" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/55cfa2e8-0b05-487b-aa8b-20573c86dbf4" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/1c7f82c3-ab02-4ede-ac12-97d5db21704d" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a1085f37-d8ad-4f8a-a32b-3781484164d8" />



## What I Learned

- Designing reusable, parameterized pipelines instead of one pipeline per file
- Building Bronze / Silver / Gold layers with Mapping Data Flows
- Orchestrating multiple pipelines and sending automated alerts

## Author

**Muneer** | [GitHub](https://github.com/muneer45-hub) | www.linkedin.com/in/shaik-mohammed-muneer-629b7b290
