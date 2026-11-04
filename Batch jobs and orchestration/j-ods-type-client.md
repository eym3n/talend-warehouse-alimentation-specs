# ODS Customer Type Loader (jOdsTypeClient)

## Purpose and Scope

The `jOdsTypeClient` job (version `0.1`, item path `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/ODS/jOdsTypeClient_0.1.item`) extracts customer account classification definitions from daily delimited flat files (`ICOM_*_TYPE_CLIENT.csv`), enforces mandatory code and label completeness, quarantines duplicates, and manages SCD Type 2 active versioning in `VENTE_ODS.ODS_TYPE_CLIENT`.

```mermaid
sequenceDiagram
    autonumber
    actor Caller as jChargeODS
    participant Job as jOdsTypeClient
    participant FS as Ingress_Files
    participant Memory as Engine_Memory
    participant ODS_Table as PostgreSQL_ODS_TYPE_CLIENT
    participant Rejet_Table as PostgreSQL_ODS_REJET

    Caller->>Job: Invoke jOdsTypeClient with context

    Note over Job,Memory: Stage 1 - Prejob Initialization
    Job->>Job: tChronometerStart_1 (Capture start timestamp)
    Job->>ODS_Table: Open shared connection VENTE_ODS (AutoCommit = false)
    Job->>Memory: tSetGlobalVar_1 (Set masque_fichier="ICOM_*_TYPE_CLIENT.csv", nom_flux="TYPECLIENT")
    Job->>Job: tJava_1 (Log job start datetime)

    Note over Job,FS: Stage 2 - Ingress File Discovery
    Job->>FS: tFileList_1 (Scan directory for masque_fichier)
    alt File Missing or Directory Empty (OnComponentError)
        FS-->>Job: File not found
        Job->>Job: tDie_1 (Exit Code 4 - Directory does not contain TypeClient file)
        Job-->>Caller: Terminate JVM with Exit Code 4
    else File Found (OnComponentOk)
        FS-->>Job: Locate file path
    end

    Note over Job,ODS_Table: Stage 3 - Record Iteration and Processing Loop
    Job->>FS: tFileInputDelimited_1 (Stream lines with delimiter and header)
    loop For each record line in CSV feed
        FS-->>Job: Stream raw fields (CD_TYPE_CLIENT, LB_TYPE_CLIENT)
        Job->>Memory: tMap_1 (Generate sequence NUM_LIGNE via Numeric.sequence)
        
        alt Case 1: Mandatory Fields Missing or Null (Reject Flow)
            Job->>Memory: tMap_3 (Format reject payload)
            Job->>Rejet_Table: tDBOutput_4 (INSERT into ODS_REJET - Valeurs nulles pour certain colonnes)
        else Case 2: Duplicate Business Key Detected on CD_TYPE_CLIENT (Duplicate Flow)
            Job->>Memory: tMap_4 (Format duplicate reject payload)
            Job->>Rejet_Table: tDBOutput_5 (INSERT into ODS_REJET - Donnees en double)
        else Case 3: Valid New Record (No Match in ODS)
            Job->>Memory: tMap_2 (Map new record with BL_LIGNE_ACTIVE=1 and DT_INSERTION=now)
            Job->>ODS_Table: tDBOutput_1 (INSERT new record into ODS_TYPE_CLIENT)
        else Case 4: Valid Existing Record Changed (SCD-2 Trigger)
            Job->>Memory: tMap_2 (Prepare SCD-2 versioning: expire old and insert new)
            Job->>ODS_Table: tDBOutput_3 (UPDATE existing record: set BL_LIGNE_ACTIVE=0)
            Job->>ODS_Table: tDBOutput_2 (INSERT new version: set BL_LIGNE_ACTIVE=1 and DT_INSERTION=now)
        end
    end

    Note over Job,ODS_Table: Stage 4 - Teardown and Postjob Metrics
    Job->>Job: tPostjob_1
    Job->>ODS_Table: tDBClose_1 (Close connection VENTE_ODS)
    Job->>Job: tChronometerStop_1 (Capture elapsed duration)
    Job->>Job: tJava_2 (Log received lines, rejected lines, duplicates, and formatted duration)

    Job-->>Caller: OnSubjobOk (Exit code 0)
```

---

## Technical Component Inventory

| Component ID | Component Name | Properties & Expressions | Behavioral Function |
| :--- | :--- | :--- | :--- |
| `tPrejob_1` | `tPrejob` | Default settings | Initiates initialization sequence |
| `tChronometerStart_1` | `tChronometerStart` | Key: `tChronometerStart_1_STARTTIME` | Records start timestamp in memory |
| `tDBConnection_1` | `tPostgresqlConnection` | Shared: `"VENTE_ODS"`, `AUTO_COMMIT=false` | Connects to PostgreSQL staging schema |
| `tSetGlobalVar_1` | `tSetGlobalVar` | `masque_fichier="ICOM_*_TYPE_CLIENT.csv"`<br/>`nom_flux="TYPE_CLIENT"` | Registers feed mask and flow name in global scope |
| `tFileList_1` | `tFileList` | Dir: `context.projectFolder + TalendDate.getDate("YYYYMMDD")` | Scans for daily customer type flat file |
| `tDie_1` | `tDie` | Message: `"Le repertoire ne contient pas de fichier Type Client"`, Code: `4`, Exit: `true` | Halts execution if file is missing |
| `tFileInputDelimited_1` | `tFileInputDelimited` | File: `tFileList_1_CURRENT_FILEPATH`, Delimiter: `context.fieldSeparator` | Parses delimited text rows |
| `tMap_1` | `tMap` | `NUM_LIGNE = Numeric.sequence("s1", 1, 1)` | Stamped artificial sequence number |
| `tFilterRow_1` | `tFilterRow` | `!CD_TYPE_CLIENT.isEmpty() && !LB_TYPE_CLIENT.isEmpty()` | Discards empty/null keys |
| `tMap_3` | `tMap` | Message: `"Valeurs nulles pour certain colonnes"` | Prepares filter reject audit record |
| `tDBOutput_4` | `tPostgresqlOutput` | Table: `"ODS_REJET"`, Action: `INSERT` | Appends null rejects to quarantine log |
| `tUniqRow_1` | `tUniqRow` | Key attribute: `CD_TYPE_CLIENT` | Filters in-file duplicate keys |
| `tMap_4` | `tMap` | Message: `"Donnees en double"` | Prepares duplicate reject audit record |
| `tDBOutput_5` | `tPostgresqlOutput` | Table: `"ODS_REJET"`, Action: `INSERT` | Appends duplicate rejects to quarantine log |
| `tDBInput_1` | `tPostgresqlInput` | Query: `SELECT * FROM "VENTE_ODS"."ODS_TYPE_CLIENT"` | Feeds active lookup records to tMap_2 |
| `tMap_2` | `tMap` | Multi-output split: `insertTypeClient`, `UpdateBICurrent`, `insertNewCurrent` | Evaluates SCD Type 2 state |
| `tDBOutput_1` | `tPostgresqlOutput` | Table: `"ODS_TYPE_CLIENT"`, Action: `INSERT` | Inserts brand new customer types |
| `tDBOutput_3` | `tPostgresqlOutput` | Table: `"ODS_TYPE_CLIENT"`, Action: `UPDATE` | Deactivates superseded historical versions (`BL_LIGNE_ACTIVE=0`) |
| `tDBOutput_2` | `tPostgresqlOutput` | Table: `"ODS_TYPE_CLIENT"`, Action: `INSERT` | Inserts updated active customer type version (`BL_LIGNE_ACTIVE=1`) |
| `tPostjob_1` | `tPostjob` | Default settings | Triggers post-execution hooks |
| `tDBClose_1` | `tPostgresqlClose` | Connection: `tDBConnection_1` | Gracefully closes connection socket |
| `tChronometerStop_1` | `tChronometerStop` | Standard stopwatch | Calculates elapsed runtime |
| `tJava_2` | `tJava` | Standard print stream | Prints received, rejected, and duplicate record totals |

---

## Evidence Grounding

- Item definition: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/ODS/jOdsTypeClient_0.1.item`
- Physical table DDL: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/scriptSQL_partie_1/2-script_create_table_ods.sql` (lines 13-21)
