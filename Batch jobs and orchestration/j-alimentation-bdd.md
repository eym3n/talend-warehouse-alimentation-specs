# Master Pipeline Orchestrator (jAlimentationBDD)

## Purpose and Scope

The `jAlimentationBDD` job (version `0.1`, item path `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/Orechestration/jAlimentationBDD_0.1.item`) serves as the master execution harness for the complete warehouse alimentation cycle. It provides unified transaction scoping, execution timing, shared connection lifecycle management, and strict sequential ordering between raw staging (ODS) and analytical warehousing (DWH).

```mermaid
sequenceDiagram
    autonumber
    actor Scheduler as External_Scheduler
    participant Master as jAlimentationBDD
    participant Config as PostgreSQL_PARAMS_LOG
    participant DB_ODS as PostgreSQL_VENTE_ODS
    participant DB_DWH as PostgreSQL_VENTE_DWH
    participant ODS as jChargeODS
    participant DWH as jChargeDWH

    Scheduler->>Master: Launch Job Execution (jAlimentationBDD)

    Note over Master,Config: Stage 1 - Context Initialization
    Master->>Config: Query CONTEXTE table for environment variables
    Config-->>Master: Return key-value overrides (Server, Database, Credentials)

    Note over Master,DB_DWH: Stage 2 - Prejob Initialization
    Master->>Master: tChronometerStart_1 (Capture start timestamp)
    Master->>DB_ODS: Open shared connection VENTE ODS (AutoCommit = false)
    DB_ODS-->>Master: Connection established and shared in JVM
    Master->>DB_DWH: Open shared connection VENTE DWH (AutoCommit = false)
    DB_DWH-->>Master: Connection established and shared in JVM
    Master->>Master: tJava_1 (Log startup banner)

    Note over Master,ODS: Stage 3 - ODS Staging Orchestration
    Master->>ODS: tRunJob_1 to execute jChargeODS (transmit context)
    ODS->>ODS: Run daily directory check and full ODS ingestion pipeline
    
    alt ODS Ingestion Failed (Exit code 4 or component error)
        ODS-->>Master: Propagate Error (DIE_ON_CHILD_ERROR = true)
        Note over Master: Pipeline execution halted (jChargeDWH skipped)
    else ODS Ingestion Succeeded (OnSubjobOk)
        ODS-->>Master: Return exit code 0
        Note over Master,DWH: Stage 4 - DWH Mart Orchestration
        Master->>DWH: tRunJob_2 to execute jChargeDWH (transmit context)
        DWH->>DWH: Run dimension updates and fact sales aggregation
        DWH-->>Master: Return exit code 0 on success
    end

    Note over Master,DB_DWH: Stage 5 - Postjob Teardown and Cleanup
    Master->>DB_ODS: tDBClose_1 (Close connection VENTE ODS)
    Master->>DB_DWH: tDBClose_2 (Close connection VENTE DWH)
    Master->>Master: tChronometerStop_1 (Compute elapsed duration)
    Master->>Master: tJava_2 (Format duration via customCode.getFormattedTime)
```

---

## Component Topology and Parameters

| Component ID | Component Type | Configuration & Technical Properties | Functional Role |
| :--- | :--- | :--- | :--- |
| `tPrejob_1` | `tPrejob` | Offset: (96, 64) | Triggers pre-execution initialization flow |
| `tChronometerStart_1` | `tChronometerStart` | Caption: `""` | Captures start epoch milliseconds into `globalMap` (`tChronometerStart_1_STARTTIME`) |
| `tDBConnection_1` | `tPostgresqlConnection` | Schema: `context.schema_ods`, Shared: `"VENTE ODS"`, `AUTO_COMMIT=false` | Opens connection to staging schema in `vente_db` |
| `tDBConnection_2` | `tPostgresqlConnection` | Schema: `context.schema_dwh`, Shared: `"VENTE DWH"`, `AUTO_COMMIT=false` | Opens connection to analytical mart schema in `vente_db` |
| `tJava_1` | `tJava` | Standard output print stream | Emits startup banner and formatted start datetime |
| `tRunJob_1` | `tRunJob` | Process: `jChargeODS`, `DIE_ON_CHILD_ERROR=true`, `TRANSMIT_WHOLE_CONTEXT=true` | Invokes ODS ingestion pipeline synchronously |
| `tRunJob_2` | `tRunJob` | Process: `jChargeDWH`, `DIE_ON_CHILD_ERROR=true`, `TRANSMIT_WHOLE_CONTEXT=true` | Invokes DWH dimensional mart pipeline synchronously |
| `tPostjob_1` | `tPostjob` | Offset: (96, 480) | Triggers teardown hooks regardless of standard flow outcome |
| `tDBClose_1` | `tPostgresqlClose` | Connection: `tDBConnection_1` | Closes connection to `VENTE_ODS` |
| `tDBClose_2` | `tPostgresqlClose` | Connection: `tDBConnection_2` | Closes connection to `VENTE_DWH` |
| `tChronometerStop_1` | `tChronometerStop` | Since Beginning: `true` | Computes elapsed time into `globalMap.get("tChronometerStop_1_DURATION")` |
| `tJava_2` | `tJava` | Standard output print stream | Emits completion banner and formatted execution duration string |

---

## Control Flow and Error Handling

1. **Failure Propagation:**
   - Both child jobs configure `DIE_ON_CHILD_ERROR = true`. If any component in `jChargeODS` fails (e.g. missing daily directory or schema violation), `jAlimentationBDD` terminates immediately with a non-zero exit code, preventing `jChargeDWH` from running on partial or corrupted staging data.
2. **Postjob Guarantees:**
   - The `tPostjob` sequence executes even on JVM shutdown signals or child errors, ensuring that database connection sockets are closed and resources freed.
3. **Implicit Context Loading:**
   - Inherits implicit database context loader reading table `PARAMS_LOG.CONTEXTE`. Values fetched override design-time defaults for `ServerName`, `database`, `port`, `utilisateur`, and `password`.

---

## Evidence Grounding

- Source definition: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/Orechestration/jAlimentationBDD_0.1.item`
- Context parameters: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/context/projectvariable_0.1.item`
- Formatting routine: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/customCode.java`
