# ODS Batch Ingestion Orchestrator (jChargeODS)

## Purpose and Scope

The `jChargeODS` job (version `0.1`, item path `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/Orechestration/jChargeODS_0.1.item`) controls the extraction of raw daily CSV feeds and feeds them into the Operational Data Store. It enforces strict relational dependency sequencing across dimensions and transactional facts, verifying file presence before loading any tables.

```mermaid
sequenceDiagram
    autonumber
    actor Caller as jAlimentationBDD_Operator
    participant Orch as jChargeODS
    participant FS as File_System_context.projectFolder
    participant DB as PostgreSQL_VENTE_ODS
    participant J_CAT as jOdsCategorie
    participant J_SCAT as jOdsSousCategorie
    participant J_PROD as jOdsProduit
    participant J_TCLI as jOdsTypeClient
    participant J_CLI as jOdsClient
    participant J_VTE as jOdsVente

    Caller->>Orch: Invoke jChargeODS with context

    Note over Orch,FS: Stage 1 - Prejob and Directory Gating
    Orch->>Orch: tChronometerStart_1 (Capture start epoch)
    Orch->>FS: tFileList_1 (Scan directory: context.projectFolder + YYYYMMDD)
    alt Directory Missing (OnComponentError)
        FS-->>Orch: Directory not found
        Orch->>Orch: tDie_1 (Exit Code 4, "le repertoire du jour n'existe pas", EXIT_JVM=true)
        Orch-->>Caller: Process Aborted with Exit Code 4
    else Directory Found (OnComponentOk)
        FS-->>Orch: Directory confirmed
        Orch->>DB: Open shared connection "VENTE_ODS" (AutoCommit = false)
        DB-->>Orch: Connection ready
        Orch->>Orch: tJava_1 (Log startup message)
    end

    Note over Orch,J_VTE: Stage 2 - Sequential Relational Ingestion Chain
    Orch->>J_CAT: tRunJob_1 (Execute jOdsCategorie)
    J_CAT->>J_CAT: Ingest ICOM_*_CATEGORY.csv into ODS_CATEGORIE
    J_CAT-->>Orch: OnSubjobOk (Category load complete)

    Orch->>J_SCAT: tRunJob_2 (Execute jOdsSousCategorie)
    J_SCAT->>J_SCAT: Ingest ICOM_*_SOUS_CATEGORIE.csv into ODS_SOUS_CATEGORIE
    J_SCAT-->>Orch: OnSubjobOk (Subcategory load complete)

    Orch->>J_PROD: tRunJob_3 (Execute jOdsProduit)
    J_PROD->>J_PROD: Ingest ICOM_*_PRODUIT.csv into ODS_PRODUIT
    J_PROD-->>Orch: OnSubjobOk (Product load complete)

    Orch->>J_TCLI: tRunJob_4 (Execute jOdsTypeClient)
    J_TCLI->>J_TCLI: Ingest ICOM_*_TYPE_CLIENT.csv into ODS_TYPE_CLIENT
    J_TCLI-->>Orch: OnSubjobOk (Client type load complete)

    Orch->>J_CLI: tRunJob_5 (Execute jOdsClient)
    J_CLI->>J_CLI: Ingest ICOM_*_CUSTOMER.csv into ODS_CLIENT
    J_CLI-->>Orch: OnSubjobOk (Client load complete)

    Orch->>J_VTE: tRunJob_6 (Execute jOdsVente)
    J_VTE->>J_VTE: Ingest ICOM_*_VENTE.csv into ODS_VENTE (BL_LIGNE_IN_DWH = false)
    J_VTE-->>Orch: OnSubjobOk (Sales load complete)

    Note over Orch,DB: Stage 3 - Postjob Teardown and Cleanup
    Orch->>DB: tDBClose_1 (Close connection "VENTE_ODS")
    Orch->>Orch: tChronometerStop_1 (Compute elapsed time)
    Orch->>Orch: tJava_2 (Format duration via customCode.getFormattedTime and log summary)

    Orch-->>Caller: OnSubjobOk (ODS Staging Successful)
```

---

## Daily Directory Ingress Gate

Before any database operations occur, `tFileList_1` scans `context.projectFolder` looking for a directory matching `TalendDate.getDate("YYYYMMDD")`.

- **Listing Mode:** `DIRECTORIES`, `CASE_SENSITIVE=YES`, `ERROR=true`.
- **Failure Reaction:** If the directory is absent or empty, the `OnComponentError` trigger fires `tDie_1`.
- **Termination Parameters:** Message `"le repertoire du jour n'existe pas"`, Exit Code `4`, Priority `5`, `EXIT_JVM=true`.
- **Operational Impact:** This gate guarantees that jobs will never run against missing daily data or overwrite ODS tables with empty inputs.

---

## Dependency Sequence Rationale

The subjob execution order reflects strict entity relationship constraints:

1. **`jOdsCategorie`:** Loads parent product categories (`CD_CATEGORIE`).
2. **`jOdsSousCategorie`:** Loads sub-categories which depend on valid parent category references (`CD_CATEGORIE`).
3. **`jOdsProduit`:** Loads products which depend on valid sub-category references (`CD_SOUS_CATEGORIE`).
4. **`jOdsTypeClient`:** Loads customer classification tiers (`CD_TYPE_CLIENT`).
5. **`jOdsClient`:** Loads individual customer profiles which depend on valid customer type classifications (`CD_TYPE_CLIENT`).
6. **`jOdsVente`:** Ingests granular sales transactions referencing both products (`CD_PRODUIT`) and customers (`ID_CLIENT`).

Every `tRunJob` child invocation sets `DIE_ON_CHILD_ERROR=true` and `TRANSMIT_WHOLE_CONTEXT=true`.

---

## Evidence Grounding

- Item definition: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/Orechestration/jChargeODS_0.1.item`
- Context parameters: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/context/ConnectionODS_0.1.item`
