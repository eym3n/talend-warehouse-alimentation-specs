# ODS Customer Profile Loader (jOdsClient)

## Purpose and Scope

The `jOdsClient` job (version `0.1`, item path `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/ODS/jOdsClient_0.1.item`) extracts individual customer master records from daily delimited flat files (`ICOM_*_CUSTOMER.csv`), enforces demographic and territorial data quality checks, quarantines bad records, and maintains active versions in `VENTE_ODS.ODS_CLIENT`.

```mermaid
sequenceDiagram
    autonumber
    actor Caller as jChargeODS_Orchestrator
    participant Job as jOdsClient
    participant FS as File_System_Daily_Ingress
    participant Memory as Engine_Memory_globalMap_tMap
    participant ODS_Table as PostgreSQL_ODS_CLIENT
    participant Rejet_Table as PostgreSQL_ODS_REJET

    Caller->>Job: Invoke jOdsClient with context

    Note over Job,Memory: Stage 1 - Prejob Initialization
    Job->>Job: tChronometerStart_1 (Capture start timestamp)
    Job->>ODS_Table: Open shared connection "VENTE_ODS" (AutoCommit = false)
    Job->>Memory: tSetGlobalVar_1 (Set masque_fichier="ICOM_*_CUSTOMER.csv", nom_flux="CLIENT")
    Job->>Job: tJava_1 (Log job start datetime)

    Note over Job,FS: Stage 2 - Ingress File Discovery
    Job->>FS: tFileList_1 (Scan directory for masque_fichier)
    alt File Missing or Directory Empty (OnComponentError)
        FS-->>Job: File not found
        Job->>Job: tDie_1 (Exit Code 4, "Le repertoire ne contient pas de fichier Client")
        Job-->>Caller: Terminate JVM with Exit Code 4
    else File Found (OnComponentOk)
        FS-->>Job: Locate file (tFileList_1_CURRENT_FILEPATH)
    end

    Note over Job,ODS_Table: Stage 3 - Record Iteration and Processing Loop
    Job->>FS: tFileInputDelimited_1 (Stream lines, delimiter="|", header=1)
    loop For each record line in CSV feed
        FS-->>Job: Stream raw fields (ID_CLIENT, NOM_CLIENT, PREN_CLIENT, VILLE_CLIENT, etc.)
        Job->>Memory: tMap_1 (Generate sequence NUM_LIGNE via Numeric.sequence)
        
        Job->>Job: tFilterRow_1 (Validate mandatory ID, names, city, country, region, and type)
        alt Mandatory Fields Missing or Null (Reject Flow)
            Job->>Memory: tMap_3 (Format reject payload: path, filename, line, payload, jobName)
            Job->>Rejet_Table: tDBOutput_4 (INSERT into ODS_REJET, message="Valeurs nulles pour certain colonnes")
        else Mandatory Fields Valid (Filter Flow)
            Job->>Job: tUniqRow_1 (Evaluate business key uniqueness on ID_CLIENT)
            alt Duplicate Key Detected (Duplicate Flow)
                Job->>Memory: tMap_4 (Format duplicate reject payload)
                Job->>Rejet_Table: tDBOutput_5 (INSERT into ODS_REJET, message="Donnees en double")
            else Unique Key Confirmed (Unique Flow)
                Job->>ODS_Table: tDBInput_1 (Lookup existing record by ID_CLIENT where BL_LIGNE_ACTIVE=1)
                alt No Existing Match Found in ODS
                    Job->>Memory: tMap_2 (Map new record with BL_LIGNE_ACTIVE=1, DT_INSERTION=now)
                    Job->>ODS_Table: tDBOutput_1 (INSERT new record into ODS_CLIENT)
                else Existing Record Match Found with Changed Attributes
                    Job->>Memory: tMap_2 (Prepare SCD-2 versioning: expire old, create new)
                    Job->>ODS_Table: tDBOutput_3 (UPDATE existing record: set BL_LIGNE_ACTIVE=0)
                    Job->>ODS_Table: tDBOutput_2 (INSERT new version: BL_LIGNE_ACTIVE=1, DT_INSERTION=now)
                else Existing Record Match with Identical Attributes
                    Job->>Memory: tMap_2 (No operation: active version preserved)
                end
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

## Technical Component Inventory

| Component ID | Component Name | Properties & Expressions | Behavioral Function |
| :--- | :--- | :--- | :--- |
| `tPrejob_1` | `tPrejob` | Default settings | Initiates initialization sequence |
| `tChronometerStart_1` | `tChronometerStart` | Key: `tChronometerStart_1_STARTTIME` | Records start timestamp in memory |
| `tDBConnection_1` | `tPostgresqlConnection` | Shared: `"VENTE_ODS"`, `AUTO_COMMIT=false` | Connects to PostgreSQL staging schema |
| `tSetGlobalVar_1` | `tSetGlobalVar` | `masque_fichier="ICOM_*_CUSTOMER.csv"`<br/>`nom_flux="CLIENT"` | Registers feed mask and flow name in global scope |
| `tFileList_1` | `tFileList` | Dir: `context.projectFolder + TalendDate.getDate("YYYYMMDD")` | Scans for daily customer flat file |
| `tDie_1` | `tDie` | Message: `"Le repertoire ne contient pas de fichier Client"`, Code: `4`, Exit: `true` | Halts execution if file is missing |
| `tFileInputDelimited_1` | `tFileInputDelimited` | File: `tFileList_1_CURRENT_FILEPATH`, Delimiter: `context.fieldSeparator` | Parses delimited text rows |
| `tMap_1` | `tMap` | `NUM_LIGNE = Numeric.sequence("s1", 1, 1)` | Stamped artificial sequence number |
| `tFilterRow_1` | `tFilterRow` | Non-empty check on ID, names, address, and client type | Discards empty/null keys |
| `tMap_3` | `tMap` | Message: `"Valeurs nulles pour certain colonnes"` | Prepares filter reject audit record |
| `tDBOutput_4` | `tPostgresqlOutput` | Table: `"ODS_REJET"`, Action: `INSERT` | Appends null rejects to quarantine log |
| `tUniqRow_1` | `tUniqRow` | Key attribute: `ID_CLIENT` | Filters in-file duplicate keys |
| `tMap_4` | `tMap` | Message: `"Donnees en double"` | Prepares duplicate reject audit record |
| `tDBOutput_5` | `tPostgresqlOutput` | Table: `"ODS_REJET"`, Action: `INSERT` | Appends duplicate rejects to quarantine log |
| `tDBInput_1` | `tPostgresqlInput` | Query: `SELECT * FROM "VENTE_ODS"."ODS_CLIENT"` | Feeds active lookup records to tMap_2 |
| `tMap_2` | `tMap` | Multi-output split: `insertClient`, `UpdateBICurrent`, `insertNewCurrent` | Evaluates SCD Type 2 state |
| `tDBOutput_1` | `tPostgresqlOutput` | Table: `"ODS_CLIENT"`, Action: `INSERT` | Inserts brand new customers |
| `tDBOutput_3` | `tPostgresqlOutput` | Table: `"ODS_CLIENT"`, Action: `UPDATE` | Deactivates superseded historical versions (`BL_LIGNE_ACTIVE=0`) |
| `tDBOutput_2` | `tPostgresqlOutput` | Table: `"ODS_CLIENT"`, Action: `INSERT` | Inserts updated active customer version (`BL_LIGNE_ACTIVE=1`) |
| `tPostjob_1` | `tPostjob` | Default settings | Triggers post-execution hooks |
| `tDBClose_1` | `tPostgresqlClose` | Connection: `tDBConnection_1` | Gracefully closes connection socket |
| `tChronometerStop_1` | `tChronometerStop` | Standard stopwatch | Calculates elapsed runtime |
| `tJava_2` | `tJava` | Standard print stream | Prints received, rejected, and duplicate record totals |

---

## Evidence Grounding

- Item definition: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/ODS/jOdsClient_0.1.item`
- Physical table DDL: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/scriptSQL_partie_1/2-script_create_table_ods.sql` (lines 76-91)
