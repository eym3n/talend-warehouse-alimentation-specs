# ODS Sales Transaction Ingestion (jOdsVente)

## Purpose and Scope

The `jOdsVente` job (version `0.1`, item path `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/ODS/jOdsVente_0.1.item`) ingests transactional sales data from daily files matching `ICOM_*_VENTE.csv` into `VENTE_ODS.ODS_VENTE`.

Unlike dimension loaders, sales transactions are factual ledger events rather than mutable master records. In addition to standard field validation and deduplication, `jOdsVente` initializes the warehouse synchronization watermark column `BL_LIGNE_IN_DWH = false`.

```mermaid
sequenceDiagram
    autonumber
    actor Caller as jChargeODS_Orchestrator
    participant Job as jOdsVente
    participant FS as File_System_Daily_Ingress
    participant Memory as Engine_Memory_globalMap_tMap
    participant ODS_Table as PostgreSQL_ODS_VENTE
    participant Rejet_Table as PostgreSQL_ODS_REJET

    Caller->>Job: Invoke jOdsVente with context

    Note over Job,Memory: Stage 1 - Prejob Initialization
    Job->>Job: tChronometerStart_1 (Capture start timestamp)
    Job->>ODS_Table: Open shared connection "VENTE_ODS" (AutoCommit = false)
    Job->>Memory: tSetGlobalVar_1 (Set masque_fichier="ICOM_*_VENTE.csv", nom_flux="VENTE")
    Job->>Job: tJava_1 (Log job start datetime)

    Note over Job,FS: Stage 2 - Ingress File Discovery
    Job->>FS: tFileList_1 (Scan directory for masque_fichier)
    alt File Missing or Directory Empty (OnComponentError)
        FS-->>Job: File not found
        Job->>Job: tDie_1 (Exit Code 4, "Le repertoire ne contient pas de fichier Vente")
        Job-->>Caller: Terminate JVM with Exit Code 4
    else File Found (OnComponentOk)
        FS-->>Job: Locate file (tFileList_1_CURRENT_FILEPATH)
    end

    Note over Job,ODS_Table: Stage 3 - Record Iteration and Processing Loop
    Job->>FS: tFileInputDelimited_1 (Stream lines, delimiter="|", header=1)
    loop For each sales line item in CSV feed
        FS-->>Job: Stream raw fields (ID_VENTE, CD_PRODUIT, DT_VENTE, ID_CLIENT, QTE_VENTE, PRIX_ACHAT, PRIX_VENTE)
        Job->>Memory: tMap_1 (Generate sequence NUM_LIGNE via Numeric.sequence)
        
        Job->>Job: tFilterRow_1 (Validate all key identifiers, date, quantity, and prices non-null)
        alt Mandatory Sales Fields Missing or Null (Reject Flow)
            Job->>Memory: tMap_3 (Format reject payload: path, filename, line, payload, jobName)
            Job->>Rejet_Table: tDBOutput_2 (INSERT into ODS_REJET, message="Valeurs nulles pour certain colonnes")
        else Mandatory Sales Fields Valid (Filter Flow)
            Job->>Job: tUniqRow_1 (Evaluate composite uniqueness on ID_VENTE + CD_PRODUIT)
            alt Duplicate Sales Line Detected (Duplicate Flow)
                Job->>Memory: tMap_4 (Format duplicate reject payload)
                Job->>Rejet_Table: tDBOutput_3 (INSERT into ODS_REJET, message="Donnees en double")
            else Unique Line Item Confirmed (Unique Flow)
                Job->>Memory: tMap_2 (Map ODS columns and set BL_LIGNE_IN_DWH = false, DT_INSERTION = now)
                Job->>ODS_Table: tDBOutput_1 (INSERT staged transaction into ODS_VENTE)
            end
        end
    end

    Note over Job,ODS_Table: Stage 4 - Teardown and Postjob Metrics
    Job->>Job: tPostjob_1
    Job->>ODS_Table: tDBClose_1 (Close connection)
    Job->>Job: tChronometerStop_1 (Capture elapsed duration)
    Job->>Job: tJava_2 (Log received lines, rejected lines, duplicates, and total formatted duration)

    Job-->>Caller: OnSubjobOk (Exit code 0)
```

---

## Technical Transformation Details

1. **Input Schema:**
   - `ID_VENTE` (String)
   - `CD_PRODUIT` (String)
   - `DT_VENTE` (Date, pattern `"dd-MM-yyyy"`)
   - `ID_CLIENT` (String)
   - `QTE_VENTE` (Integer)
   - `PRIX_ACHAT` (Numeric / Double)
   - `PRIX_VENTE` (Numeric / Double)
2. **Key Uniqueness Rule:**
   - The composite primary key across the sales file is `ID_VENTE` + `CD_PRODUIT` (a transaction order line item). `tUniqRow_1` flags both columns as key attributes.
3. **Audit and Watermarking Attributes:**
   - `LB_NOM_FICHIER`: current file name from `((String)globalMap.get("tFileList_1_CURRENT_FILE"))`
   - `DT_INSERTION`: current execution timestamp (`TalendDate.getCurrentDate()`)
   - `LB_JOB_NAME`: execution job name (`jobName`)
   - `BL_LIGNE_IN_DWH`: initialized as boolean literal `false`. This flag indicates to the downstream DWH pipeline that this transaction line is pending ingestion into `FAIT_VENTE`.

---

## Evidence Grounding

- Item definition: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/ODS/jOdsVente_0.1.item`
- Physical DDL: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/scriptSQL_partie_1/2-script_create_table_ods.sql`
