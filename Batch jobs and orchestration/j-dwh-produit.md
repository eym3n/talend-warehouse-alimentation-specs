# DWH Product Dimension Loader (jDwhProduit)

## Purpose and Scope

The `jDwhProduit` job (version `0.1`, item path `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/DWH/jDwhProduit_0.1.item`) extracts product reference records from the ODS staging layer, denormalizes the complete category and sub-category hierarchy, and maintains the conformed product dimension `VENTE_DWH.DIM_PRODUIT` using Slowly Changing Dimension Type 2 (SCD-2) versioning.

```mermaid
sequenceDiagram
    autonumber
    actor Caller as jChargeDWH_Orchestrator
    participant Job as jDwhProduit
    participant ODS_P as PostgreSQL_ODS_PRODUIT
    participant ODS_SC as PostgreSQL_ODS_SOUS_CATEGORIE
    participant ODS_C as PostgreSQL_ODS_CATEGORIE
    participant DWH_P as PostgreSQL_DIM_PRODUIT
    participant Memory as Engine_Memory_tMap

    Caller->>Job: Invoke jDwhProduit with context

    Note over Job,DWH_P: Stage 1 - Prejob Connections
    Job->>Job: tChronometerStart_1 (Capture start epoch)
    Job->>ODS_P: Open shared connection VENTE_ODS (AutoCommit = false)
    Job->>DWH_P: Open shared connection VENTE_DWH (AutoCommit = false)
    Job->>Job: tJava_1 (Log job start banner)

    Note over Job,ODS_C: Stage 2 - Hierarchical Denormalization
    Job->>ODS_P: tDBInput_1 (Query active products: WHERE BL_LIGNE_ACTIVE = 1)
    Job->>ODS_SC: tDBInput_2 (Lookup active subcategories on CD_SOUS_CATEGORIE)
    Job->>ODS_C: tDBInput_3 (Lookup active categories on CD_CATEGORIE)
    Memory-->>Job: Flatten hierarchy into unified Product record (CD_PRODUIT, NOM_PRODUIT, PRIX_*, LB_SOUS_CATEGORIE, LB_CATEGORIE)

    Note over Job,DWH_P: Stage 3 - SCD Type 2 Evaluation and Ingestion Loop
    Job->>DWH_P: tDBInput_4 (Lookup current active dimension records: WHERE BL_LIGNE_ACTIVE = 1)
    loop For each denormalized product record
        Job->>Memory: tMap_2 (Compare incoming attributes against active DIM_PRODUIT lookup on CD_PRODUIT)
        
        alt No Match Found in DIM_PRODUIT - Brand New Product
            Job->>Memory: Prepare initial version (BL_LIGNE_ACTIVE = 1, DT_INSERTION = now, DT_MISE_A_JOUR = null)
            Job->>DWH_P: tDBOutput_1 (INSERT new row using sequence DIM_PRODUIT_seq)
        else Match Found and Attributes Changed - SCD-2 Trigger
            Note over Job,DWH_P: Change detected in NOM_PRODUIT, prices, or category hierarchy
            Job->>Memory: Invalidate prior version and prepare new active version
            Job->>DWH_P: tDBOutput_2 (UPDATE existing row: set BL_LIGNE_ACTIVE = 0, DT_MISE_A_JOUR = now)
            Job->>DWH_P: tDBOutput_3 (INSERT new active row: set BL_LIGNE_ACTIVE = 1, DT_INSERTION = now, DT_MISE_A_JOUR = null)
        else Match Found and Attributes Identical
            Job->>Memory: Active version preserved unchanged (No write)
        end
    end

    Note over Job,DWH_P: Stage 4 - Teardown and Postjob Metrics
    Job->>Job: tPostjob_1
    Job->>ODS_P: tDBClose_1 (Close connection VENTE_ODS)
    Job->>DWH_P: tDBClose_2 (Close connection VENTE_DWH)
    Job->>Job: tChronometerStop_1 (Compute elapsed duration)
    Job->>Job: tJava_2 (Format duration via customCode.getFormattedTime and log summary)

    Job-->>Caller: OnSubjobOk (Product dimension updated)
```

---

## SCD Type 2 Comparison Attributes

A new version is triggered in `DIM_PRODUIT` whenever any of the following attributes change compared to the current active record:
- `NOM_PRODUIT`: Commercial product name.
- `PRIX_ACHAT_PRODUIT`: Unit purchase cost.
- `PRIX_VENTE_PRODUIT`: Suggested retail price.
- `CD_SOUS_CATEGORIE`: Sub-category identifier.
- `LB_SOUS_CATEGORIE`: Sub-category descriptive title.
- `CD_CATEGORIE`: Category identifier.
- `LB_CATEGORIE`: Category descriptive title.

---

## Technical Component Inventory

| Component ID | Component Name | Properties & Expressions | Behavioral Function |
| :--- | :--- | :--- | :--- |
| `tPrejob_1` | `tPrejob` | Default settings | Initiates initialization sequence |
| `tChronometerStart_1` | `tChronometerStart` | Key: `tChronometerStart_1_STARTTIME` | Records start timestamp in memory |
| `tDBConnection_1` | `tPostgresqlConnection` | Shared: `"VENTE_ODS"`, `AUTO_COMMIT=false` | Connects to PostgreSQL staging schema |
| `tDBConnection_2` | `tPostgresqlConnection` | Shared: `"VENTE_DWH"`, `AUTO_COMMIT=false` | Connects to PostgreSQL warehouse schema |
| `tDBInput_1` | `tPostgresqlInput` | `SELECT * FROM "VENTE_ODS"."ODS_PRODUIT" WHERE "BL_LIGNE_ACTIVE" = 1` | Primary extraction stream |
| `tDBInput_2` | `tPostgresqlInput` | `SELECT * FROM "VENTE_ODS"."ODS_SOUS_CATEGORIE" WHERE "BL_LIGNE_ACTIVE" = 1` | Lookup subcategories |
| `tDBInput_3` | `tPostgresqlInput` | `SELECT * FROM "VENTE_ODS"."ODS_CATEGORIE" WHERE "BL_LIGNE_ACTIVE" = 1` | Lookup categories |
| `tMap_1` | `tMap` | Inner joins on subcategory and category codes | Flattens hierarchical structures |
| `tDBInput_4` | `tPostgresqlInput` | `SELECT * FROM "VENTE_DWH"."DIM_PRODUIT" WHERE "BL_LIGNE_ACTIVE" = 1` | Active dimension lookup stream |
| `tMap_2` | `tMap` | Attribute comparator: detects attribute changes | Directs SCD-2 version routing |
| `tDBOutput_1` | `tPostgresqlOutput` | Table: `"DIM_PRODUIT"`, Action: `INSERT` | Appends new initial product versions |
| `tDBOutput_2` | `tPostgresqlOutput` | Table: `"DIM_PRODUIT"`, Action: `UPDATE` | Deactivates superseded rows (`BL_LIGNE_ACTIVE=0`) |
| `tDBOutput_3` | `tPostgresqlOutput` | Table: `"DIM_PRODUIT"`, Action: `INSERT` | Appends updated active product versions |
| `tPostjob_1` | `tPostjob` | Default settings | Triggers post-execution hooks |
| `tDBClose_1` | `tPostgresqlClose` | Connection: `tDBConnection_1` | Closes connection to `VENTE_ODS` |
| `tDBClose_2` | `tPostgresqlClose` | Connection: `tDBConnection_2` | Closes connection to `VENTE_DWH` |
| `tChronometerStop_1` | `tChronometerStop` | Standard stopwatch | Calculates elapsed runtime |
| `tJava_2` | `tJava` | Standard print stream | Prints completion duration banner |

---

## Evidence Grounding

- Item definition: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/DWH/jDwhProduit_0.1.item`
- Physical DDL: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/scriptSQL_partie_2/2-script_create_table_dwh.sql` (lines 13-30)
