# DWH Customer Dimension Loader (jDwhClient)

## Purpose and Scope

The `jDwhClient` job (version `0.1`, item path `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/DWH/jDwhClient_0.1.item`) extracts individual customer master records from the ODS layer, denormalizes customer type classifications, and maintains conformed customer dimension records in `VENTE_DWH.DIM_CLIENT` using Slowly Changing Dimension Type 2 (SCD-2) versioning.

```mermaid
sequenceDiagram
    autonumber
    actor Caller as jChargeDWH_Orchestrator
    participant Job as jDwhClient
    participant ODS_CLI as PostgreSQL_ODS_CLIENT
    participant ODS_TCLI as PostgreSQL_ODS_TYPE_CLIENT
    participant DWH_CLI as PostgreSQL_DIM_CLIENT
    participant Memory as Engine_Memory_tMap

    Caller->>Job: Invoke jDwhClient with context

    Note over Job,DWH_CLI: Stage 1 - Prejob Connections
    Job->>Job: tChronometerStart_1 (Capture start epoch)
    Job->>ODS_CLI: Open shared connection "VENTE_ODS" (AutoCommit = false)
    Job->>DWH_CLI: Open shared connection "VENTE_DWH" (AutoCommit = false)
    Job->>Job: tJava_1 (Log job start banner)

    Note over Job,ODS_TCLI: Stage 2 - Customer Tier Denormalization
    Job->>ODS_CLI: tDBInput_1 (Query active customers: WHERE BL_LIGNE_ACTIVE = 1)
    Job->>ODS_TCLI: tDBInput_2 (Lookup active customer types on CD_TYPE_CLIENT)
    Memory-->>Job: Flatten customer demographic and classification attributes

    Note over Job,DWH_CLI: Stage 3 - SCD Type 2 Evaluation and Ingestion Loop
    Job->>DWH_CLI: tDBInput_3 (Lookup current active dimension records: WHERE BL_LIGNE_ACTIVE = 1)
    loop For each denormalized customer record
        Job->>Memory: tMap_2 (Compare incoming attributes against active DIM_CLIENT lookup on ID_CLIENT)
        
        alt No Match Found in DIM_CLIENT (Brand New Customer)
            Job->>Memory: Prepare initial version (BL_LIGNE_ACTIVE = 1, DT_INSERTION = now, DT_MISE_A_JOUR = null)
            Job->>DWH_CLI: tDBOutput_1 (INSERT new row using sequence DIM_CLIENT_seq)
        else Match Found and Attributes Changed (SCD-2 Trigger)
            Note over Job,DWH_CLI: Change detected in address, city, country, region, or account classification
            Job->>Memory: Invalidate prior version and prepare new active version
            Job->>DWH_CLI: tDBOutput_2 (UPDATE existing row: set BL_LIGNE_ACTIVE = 0, DT_MISE_A_JOUR = now)
            Job->>DWH_CLI: tDBOutput_3 (INSERT new active row: set BL_LIGNE_ACTIVE = 1, DT_INSERTION = now, DT_MISE_A_JOUR = null)
        else Match Found and Attributes Identical
            Job->>Memory: Active version preserved unchanged (No write)
        end
    end

    Note over Job,DWH_CLI: Stage 4 - Teardown and Postjob Metrics
    Job->>Job: tPostjob_1
    Job->>ODS_CLI: tDBClose_1 (Close connection "VENTE_ODS")
    Job->>DWH_CLI: tDBClose_2 (Close connection "VENTE_DWH")
    Job->>Job: tChronometerStop_1 (Compute elapsed duration)
    Job->>Job: tJava_2 (Format duration via customCode.getFormattedTime and log summary)

    Job-->>Caller: OnSubjobOk (Customer dimension updated)
```

---

## SCD Type 2 Comparison Attributes

A new version is triggered in `DIM_CLIENT` whenever any of the following demographic or geographical attributes change compared to the current active record:
- `NOM_CLIENT`: Customer surname.
- `PREN_CLIENT`: Customer first name.
- `CD_POSTAL_CLIENT`: Postal code.
- `VILLE_CLIENT`: City.
- `PAYS_CLIENT`: Country.
- `REGION_CLIENT`: Administrative region / territory.
- `CD_TYPE_CLIENT`: Account classification identifier.
- `LB_TYPE_CLIENT`: Classification descriptive label.

---

## Technical Component Inventory

| Component ID | Component Name | Properties & Expressions | Behavioral Function |
| :--- | :--- | :--- | :--- |
| `tPrejob_1` | `tPrejob` | Default settings | Initiates initialization sequence |
| `tChronometerStart_1` | `tChronometerStart` | Key: `tChronometerStart_1_STARTTIME` | Records start timestamp in memory |
| `tDBConnection_1` | `tPostgresqlConnection` | Shared: `"VENTE_ODS"`, `AUTO_COMMIT=false` | Connects to PostgreSQL staging schema |
| `tDBConnection_2` | `tPostgresqlConnection` | Shared: `"VENTE_DWH"`, `AUTO_COMMIT=false` | Connects to PostgreSQL warehouse schema |
| `tDBInput_1` | `tPostgresqlInput` | `SELECT * FROM "VENTE_ODS"."ODS_CLIENT" WHERE "BL_LIGNE_ACTIVE" = 1` | Primary extraction stream |
| `tDBInput_2` | `tPostgresqlInput` | `SELECT * FROM "VENTE_ODS"."ODS_TYPE_CLIENT" WHERE "BL_LIGNE_ACTIVE" = 1` | Lookup customer types |
| `tMap_1` | `tMap` | Inner join on `CD_TYPE_CLIENT` | Denormalizes customer classification label |
| `tDBInput_3` | `tPostgresqlInput` | `SELECT * FROM "VENTE_DWH"."DIM_CLIENT" WHERE "BL_LIGNE_ACTIVE" = 1` | Active dimension lookup stream |
| `tMap_2` | `tMap` | Attribute comparator: detects address/name/tier changes | Directs SCD-2 version routing |
| `tDBOutput_1` | `tPostgresqlOutput` | Table: `"DIM_CLIENT"`, Action: `INSERT` | Appends new initial customer versions |
| `tDBOutput_2` | `tPostgresqlOutput` | Table: `"DIM_CLIENT"`, Action: `UPDATE` | Deactivates superseded rows (`BL_LIGNE_ACTIVE=0`) |
| `tDBOutput_3` | `tPostgresqlOutput` | Table: `"DIM_CLIENT"`, Action: `INSERT` | Appends updated active customer versions |
| `tPostjob_1` | `tPostjob` | Default settings | Triggers post-execution hooks |
| `tDBClose_1` | `tPostgresqlClose` | Connection: `tDBConnection_1` | Closes connection to `VENTE_ODS` |
| `tDBClose_2` | `tPostgresqlClose` | Connection: `tDBConnection_2` | Closes connection to `VENTE_DWH` |
| `tChronometerStop_1` | `tChronometerStop` | Standard stopwatch | Calculates elapsed runtime |
| `tJava_2` | `tJava` | Standard print stream | Prints completion duration banner |

---

## Evidence Grounding

- Item definition: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/DWH/jDwhClient_0.1.item`
- Physical DDL: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/scriptSQL_partie_2/2-script_create_table_dwh.sql` (lines 38-57)
