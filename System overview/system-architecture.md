# Data Warehouse Alimentation System Architecture

## Architecture Overview

The **Talend Data Warehouse Alimentation** system processes raw business data through a structured three-tier enterprise data integration pipeline:

1. **Ingress Tier (File System / Staging):** Source systems deliver date-stamped CSV files into a directory hierarchy partitioned by execution date (`YYYYMMDD`). External holiday reference data is supplied via Excel (`joursFeries.xlsx`).
2. **Operational Data Store (ODS Tier):** Normalizes, deduplicates, and validates file contents. Enforces data quality by redirecting invalid or duplicate lines to `ODS_REJET`, tracks versioning using active-flag indicators (`BL_LIGNE_ACTIVE`), and flags transactions for warehouse promotion (`BL_LIGNE_IN_DWH`).
3. **Data Warehouse (DWH Tier):** Transforms normalized ODS structures into a star schema hosted in the `VENTE_DWH` schema. Maintains Type 2 Slowly Changing Dimensions (SCD-2) for Customer and Product entities, calculates business metrics, joins transactional sales with conformed time dimensions, and populates the summarized fact table `FAIT_VENTE`.

```mermaid
graph LR
    subgraph Storage["PostgreSQL Database - vente_db"]
        subgraph S_PARAMS["Schema - PARAMS_LOG"]
            CTX["CONTEXTE<br/>(Dynamic Config)"]
        end
        subgraph S_ODS["Schema - VENTE_ODS"]
            ODS_T["ODS Tables<br/>(CATEGORY, PRODUCT, CUSTOMER, VENTE)"]
            REJ_T["ODS_REJET<br/>(Audit and Reject Log)"]
        end
        subgraph S_DWH["Schema - VENTE_DWH"]
            DIM_T["Dimensions<br/>(DIM_PRODUIT, DIM_CLIENT, DIM_CALENDRIER)"]
            FACT_T["Facts<br/>(FAIT_VENTE)"]
        end
    end

    subgraph Runtime["Talend Execution Runtime"]
        JOB_M["jAlimentationBDD"]
        JOB_O["jChargeODS"]
        JOB_D["jChargeDWH"]
    end

    FS["CSV Ingress Directory"] -->|"tFileInputDelimited"| JOB_O
    CTX -.->|"Implicit Context Load"| Runtime
    JOB_O -->|"Insert or Update"| ODS_T
    JOB_O -->|"Filter or Duplicate Reject"| REJ_T
    ODS_T -->|"Extract Active and Unloaded"| JOB_D
    JOB_D -->|"SCD Type 2"| DIM_T
    JOB_D -->|"Aggregate and Star Join"| FACT_T
    JOB_D -->|"Watermark Update"| ODS_T
```

---

## End-to-End Orchestration Sequence

The following sequence diagram illustrates the complete end-to-end operational execution order, from configuration injection through multi-job orchestration, database commits, and metric capture:

```mermaid
sequenceDiagram
    autonumber
    actor Scheduler as External_Scheduler
    participant Master as jAlimentationBDD
    participant Config as PARAMS_LOG.CONTEXTE
    participant IngressFS as Ingress_File_System
    participant ODS_Orch as jChargeODS
    participant ODS_Jobs as ODS_Loaders_Categories_>_Vente
    participant ODS_DB as PostgreSQL_VENTE_ODS
    participant DWH_Orch as jChargeDWH
    participant DWH_Jobs as DWH_Loaders_Produit_Client_Vente
    participant DWH_DB as PostgreSQL_VENTE_DWH

    Scheduler->>Master: Trigger Execution

    Note over Master,Config: Configuration Bootstrapping
    Master->>Config: Fetch active environment variables (ServerName, port, credentials)
    Config-->>Master: Inject resolved context variables

    Note over Master,ODS_DB: Connection and Monitoring Setup
    Master->>Master: tChronometerStart (Capture start epoch)
    Master->>ODS_DB: Open shared connection VENTE ODS (AutoCommit = false)
    Master->>DWH_DB: Open shared connection VENTE DWH (AutoCommit = false)

    Note over Master,ODS_DB: Stage 1 - ODS Ingestion Pipeline
    Master->>ODS_Orch: Call jChargeODS (DIE_ON_CHILD_ERROR = true)
    ODS_Orch->>IngressFS: Verify directory exists: context.projectFolder/YYYYMMDD
    alt Directory Missing
        ODS_Orch->>ODS_Orch: tDie (Exit 4: le repertoire du jour nexiste pas)
        ODS_Orch-->>Master: Abort pipeline with failure code 4
    else Directory Confirmed
        loop Sequential Entity Ingestion
            ODS_Orch->>ODS_Jobs: Execute jOdsCategorie to jOdsSousCategorie to jOdsProduit to jOdsTypeClient to jOdsClient to jOdsVente
            ODS_Jobs->>IngressFS: Read delimited rows (Pipe delimiter)
            ODS_Jobs->>ODS_DB: Filter nulls and duplicates to Log to ODS_REJET
            ODS_Jobs->>ODS_DB: Apply SCD Type 2 merges to Update ODS tables
            ODS_Jobs-->>ODS_Orch: Subjob complete
        end
        ODS_Orch-->>Master: OnSubjobOk (ODS Staging Successful)
    end

    Note over Master,DWH_DB: Stage 2 - DWH Transformation Pipeline
    Master->>DWH_Orch: Call jChargeDWH (DIE_ON_CHILD_ERROR = true)
    loop Sequential Dimensional and Fact Processing
        DWH_Orch->>DWH_Jobs: Execute jDwhProduit to jDwhClient to jDwhVente
        DWH_Jobs->>ODS_DB: Extract active master entities and unpromoted sales
        DWH_Jobs->>DWH_DB: Apply SCD Type 2 on DIM_PRODUIT and DIM_CLIENT
        DWH_Jobs->>DWH_DB: Aggregate metrics and insert into FAIT_VENTE
        DWH_Jobs->>ODS_DB: Update watermark (BL_LIGNE_IN_DWH = true)
        DWH_Jobs-->>DWH_Orch: Subjob complete
    end
    DWH_Orch-->>Master: OnSubjobOk (DWH Mart Successful)

    Note over Master,DWH_DB: Stage 3 - Transaction Commit and Teardown
    Master->>DWH_DB: Commit transaction
    Master->>ODS_DB: Commit transaction
    Master->>ODS_DB: Close connection VENTE ODS
    Master->>DWH_DB: Close connection VENTE DWH
    Master->>Master: tChronometerStop (Calculate elapsed duration)
    Master->>Master: Log formatted runtime via customCode.getFormattedTime

    Master-->>Scheduler: Execution Complete (Exit 0)
```

---

## Technical Stack and Components

| Technology Component | Version / Specification | Role in System |
| :--- | :--- | :--- |
| **ETL Engine** | Talend Open Studio for DI v8.0.1 | Visual design, code generation (Java 1.8/11), component orchestration |
| **Relational Database** | PostgreSQL 9.x / 12+ | Target operational and analytical database (`vente_db`) |
| **JDBC Driver** | `org.postgresql.Driver` (`postgresql-42.2.14.jar`) | Database connectivity across all connections |
| **Build & Packaging** | Apache Maven (`poms/pom.xml`) | Routine compilation, dependency resolution, standalone job exports |
| **In-Memory Cache** | Talend `tHashFile` (`thashfile-3.1-20190910.jar`) | In-memory record caching between fact table generation and watermark updates |
| **Custom Code** | Java 8 (`routines.customCode`) | Execution time formatting, operational logging utilities |
| **File Format** | Delimited text (`ISO-8859-15` / `UTF-8`, Pipe `\|` delimiter) | Ingress data contract for business feeds |
| **Office Automation** | Apache POI (`tFileInputExcel`) | Reference public holiday file extraction (`joursFeries.xlsx`) |

---

## Process Topology and Control Flow

The runtime orchestration follows a strict two-stage sequential workflow managed by `jAlimentationBDD`:

1. **Prejob Stage:**
   - Starts job execution stopwatch (`tChronometerStart`).
   - Initializes shared pooled connections to `VENTE_ODS` (`"VENTE ODS"`) and `VENTE_DWH` (`"VENTE DWH"`).
   - Dynamically loads configuration parameters from `PARAMS_LOG.CONTEXTE` via implicit context loading.
   - Outputs initialization banner with timestamp.
2. **ODS Extraction and Staging (`jChargeODS`):**
   - Validates that the current day's directory (`YYYYMMDD`) exists under the root path. Halts immediately with exit code `4` if absent.
   - Sequentially triggers subjobs: `jOdsCategorie` -> `jOdsSousCategorie` -> `jOdsProduit` -> `jOdsTypeClient` -> `jOdsClient` -> `jOdsVente`.
   - Each child process performs field null validation, duplicate detection, SCD versioning, and reject logging.
3. **DWH Star Schema Loading (`jChargeDWH`):**
   - Sequentially triggers dimension updates: `jDwhProduit` -> `jDwhClient`.
   - Executes fact table aggregation and loading: `jDwhVente`.
   - Updates watermark flag (`BL_LIGNE_IN_DWH = true`) in `ODS_VENTE` for processed records.
4. **Postjob Stage:**
   - Issues commits across open database transactions.
   - Closes connections gracefully.
   - Stops stopwatch and calculates total elapsed runtime using `customCode.getFormattedTime()`.

---

## Architectural Guardrails and Evidence Grounding

- **Transaction Integrity:** ODS and DWH database outputs run under explicit connection sharing with uncommitted transactions until subjobs settle and postjobs commit.
- **Watermarking:** Sales transactions in `ODS_VENTE` carry a boolean flag `BL_LIGNE_IN_DWH`. The DWH extractor exclusively queries rows where `BL_LIGNE_IN_DWH = false`.
- **SCD Type 2 Historization:** Dimension tables maintain active version flags (`BL_LIGNE_ACTIVE: 1=active, 0=historical`) and modification timestamps (`DT_MISE_A_JOUR`).
- **Codebase References:**
  - SQL Schema scripts: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/scriptSQL_partie_1/1-script_create_schema.sql`
  - Master Orchestration: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/Orechestration/jAlimentationBDD_0.1.item`
