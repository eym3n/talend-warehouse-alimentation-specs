# DWH Fact Sales Mart Aggregator (jDwhVente)

## Purpose and Scope

The `jDwhVente` job (version `0.1`, item path `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/DWH/jDwhVente_0.1.item`) is the central fact-loading engine of the warehouse. It extracts un-promoted transactional sales records from `VENTE_ODS.ODS_VENTE`, performs multi-dimensional inner lookups to conformed dimensions, computes financial metrics, aggregates transactions across common dimension keys, loads the summarized fact table `VENTE_DWH.FAIT_VENTE`, and updates the watermark flag in the ODS.

```mermaid
sequenceDiagram
    autonumber
    actor Caller as jChargeDWH_Orchestrator
    participant Job as jDwhVente
    participant ODS_V as PostgreSQL_ODS_VENTE
    participant DWH_P as PostgreSQL_DIM_PRODUIT
    participant DWH_C as PostgreSQL_DIM_CLIENT
    participant DWH_D as PostgreSQL_DIM_CALENDRIER
    participant Hash_Cache as Engine_Memory_tHashFile
    participant DWH_F as PostgreSQL_FAIT_VENTE

    Caller->>Job: Invoke jDwhVente with context

    Note over Job,DWH_F: Stage 1 - Prejob Connections
    Job->>Job: tChronometerStart_1 (Capture start epoch)
    Job->>ODS_V: Open shared connection "VENTE_ODS" (AutoCommit = false)
    Job->>DWH_F: Open shared connection "VENTE_DWH" (AutoCommit = false)
    Job->>Job: tJava_1 (Log job start datetime)

    Note over Job,Hash_Cache: Stage 2 - Extraction and Multi-Dimensional Star Join
    Job->>ODS_V: tDBInput_1 (Query unpromoted sales: WHERE BL_LIGNE_IN_DWH = false)
    ODS_V-->>Job: Stream sales records (ID_VENTE, CD_PRODUIT, DT_VENTE, ID_CLIENT, QTE, PRIX_*)
    
    Job->>DWH_P: tDBInput_7 (Lookup active product surrogate keys on CD_PRODUIT where BL_LIGNE_ACTIVE=1)
    Job->>DWH_C: tDBInput_5 (Lookup active client surrogate keys on ID_CLIENT where BL_LIGNE_ACTIVE=1)
    Job->>DWH_D: tDBInput_4 (Lookup calendar keys on DT_VENTE == DT_JOUR)
    
    loop For each unpromoted sale
        Job->>Job: tMap_1 (Compute MT_ACHAT = QTE * PRIX_ACHAT, MT_VENTE = QTE * PRIX_VENTE)
        Job->>Hash_Cache: tHashOutput_1 (Cache joined tuple and measures in memory buffer)
    end

    Note over Job,DWH_F: Stage 3 - Aggregation and Fact Table Population
    Job->>Hash_Cache: tHashInput_1 (Stream cached joined records)
    Job->>Job: tAggregateRow_1 (GROUP BY ID_DIM_PRODUIT, ID_DIM_CLIENT, ID_DIM_CALENDRIER)
    loop For each unique dimension tuple grouping
        Job->>Job: Compute aggregates: count(ID_VENTE), sum(QTE_VENTE), sum(MT_ACHAT), sum(MT_VENTE)
        Job->>DWH_F: tDBOutput_1 (INSERT summarized record into VENTE_DWH.FAIT_VENTE)
    end

    Note over Job,ODS_V: Stage 4 - Watermark Synchronization
    Job->>Hash_Cache: tHashInput_2 (Stream cached records for watermark update)
    loop For each processed transaction
        Job->>Job: tMap_3 (Set BL_LIGNE_IN_DWH = true)
        Job->>ODS_V: tDBOutput_2 (UPDATE ODS_VENTE by ID_VENTE and CD_PRODUIT)
    end

    Note over Job,DWH_F: Stage 5 - Two-Phase Commit and Teardown
    Job->>Job: tPostjob_1
    Job->>DWH_F: tDBCommit_1 (Commit VENTE_DWH transaction)
    Job->>ODS_V: tDBCommit_2 (Commit VENTE_ODS transaction)
    Job->>ODS_V: tDBClose_1 (Close connection)
    Job->>DWH_F: tDBClose_2 (Close connection)
    Job->>Job: tChronometerStop_1 (Capture elapsed duration)
    Job->>Job: tJava_2 (Format duration via customCode.getFormattedTime and log summary)

    Job-->>Caller: OnSubjobOk (Fact table aggregation complete)
```

---

## Fact Metric Formulas

For each incoming sale row in `tMap_1`:
- **`MT_ACHAT`:** `row1.QTE_VENTE * row1.PRIX_ACHAT`
- **`MT_VENTE`:** `row1.QTE_VENTE * row1.PRIX_VENTE`

In `tAggregateRow_1`, transactions are grouped by the composite dimension surrogate key `(ID_DIM_PRODUIT, ID_DIM_CLIENT, ID_DIM_CALENDRIER)` and summarized:
- **`NB_VENTE`:** `count(ID_VENTE)` (total number of distinct line items)
- **`NB_QTE_VENTE`:** `count(ID_DIM_PRODUIT)` (total units sold across the grouping)
- **`NB_MT_ACHAT`:** `sum(MT_ACHAT)` (total purchase cost)
- **`NB_MT_VENTE`:** `sum(MT_VENTE)` (total gross sales revenue)

---

## Watermark Synchronization and Atomicity

1. **In-Memory Caching (`tHashOutput_1`):** To avoid re-querying the database or losing track of the exact set of processed records, joined records are retained in RAM.
2. **Post-Aggregation Watermarking:** Once `FAIT_VENTE` insertion succeeds (`OnSubjobOk`), `tHashInput_2` streams the original processed keys (`ID_VENTE`, `CD_PRODUIT`) into `tMap_3` where `BL_LIGNE_IN_DWH = true`.
3. **Database Commit Scoping:** In `tPostjob`, `tDBCommit_1` commits the DWH transaction and `tDBCommit_2` commits the ODS transaction, guaranteeing that records are never marked as processed in ODS if fact insertion fails.

---

## Evidence Grounding

- Job design: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/ALIMENTATION_DATA_WAREHOUSE/process/DWH/jDwhVente_0.1.item`
- Fact table DDL: `workspace://codebase/7c8eded6-9911-48de-929c-053e124fc8b5/file/scriptSQL_partie_2/2-script_create_table_dwh.sql`
