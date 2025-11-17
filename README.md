# On-Premise SQL Server to Azure SQL DB Migration Project

```
[ YOUR LAPTOP / ON-PREMISE NETWORK ]             [ AZURE CLOUD ]
+-----------------------------------+             +-------------------------------------------------+
|                                   |             |                                                 |
|  [On-Prem SQL Server]             |             |  [Azure SQL Database (Sink)]                    |
|  (Source: AdventureWorks)         |             |                                                 |
|           ^                       |   HTTPS     |           ^                                     |
|           | (Pulls Data)          |  (Pushes    |           | (Writes Data)                       |
|           |                       |   Data)     |           |                                     |
|  [Self-Hosted Integration Runtime]|<------------+  [Azure Data Factory]                           |
|  (SHIR - Secure Gateway)          | (Commands)  |  (Orchestrator: PL_Initial_Load_Dynamic)        |
|                                   |             |                                                 |
+-----------------------------------+             +-------------------------------------------------+
```
    
## 1. Project Overview

This project demonstrates a complete, real-world data migration of the AdventureWorks OLTP database from an on-premise SQL Server to a cloud-native Azure SQL Database. The migration is performed with minimal downtime using a modern, metadata-driven approach with Azure Data Factory (ADF) and Change Data Capture (CDC).

The primary goal is to move beyond a simple "lift-and-shift" and build a robust, repeatable, and validatable pipeline for a live database.

* **Source System:** SQL Server 2019 (on-premise)
* **Target System:** Azure SQL Database (PaaS)
* **Core Tool:** Azure Data Factory (ADF)
* **Migration Method:** Initial Load (ADF) + Real-time Sync (CDC)

---

## 2. Key Features

* **Dynamic & Metadata-Driven:** The pipeline is not hard-coded. It uses a `Lookup` activity to fetch a list of all tables and dynamically copies them, making it scalable and easy to maintain.
* **Automated Schema Validation:** Includes a dedicated validation loop that checks row counts between the source and sink for *every table* after the copy, ensuring 100% data integrity.
* **Resilient & Idempotent:** The pipeline is designed to be re-run safely. It automatically disables constraints, truncates destination tables, and then re-enables constraints, handling both `FOREIGN KEY` and `PRIMARY KEY` violations.
* **Handles Real-World Complexity:** The pipeline dynamically handles schema differences like **computed columns**, which would cause a standard copy to fail.
* **Zero-Data-Loss Design:** By enabling **Change Data Capture (CDC)** before the initial load, the pipeline is ready for a minimal-downtime cutover. (This README covers the initial load; the next phase is the incremental sync).

---

## 3. Architecture

The solution is built on two main pipelines within Azure Data Factory:

### Pipeline 1: `PL_Initial_Load_Dynamic`
This is the main pipeline responsible for the one-time full load and validation.

**Flow:**
1.  **`Script_DisableConstraints` (Script Activity):**
    * Runs a dynamic SQL script on the target Azure SQL DB to find and disable all foreign key constraints.
2.  **`Lookup_GetTables` (Lookup Activity):**
    * Queries the on-prem SQL Server's system tables (`sys.tables`, `sys.columns`).
    * Builds a JSON array of all tables, *and* a pre-built list of all non-computed columns for each table.
3.  **`FE_OnPrem_Table` (ForEach Loop - Parallel):**
    * Loops through the table list from the `Lookup`.
    * **`Copy Data` Activity (Inside Loop):**
        * **Source:** Uses a dynamic query (`SELECT [ColumnList] FROM [Schema].[Table]`) to pull only writeable columns.
        * **Sink:** Uses a dynamic `DELETE FROM [Schema].[Table]` pre-copy script to ensure the table is empty before loading.
4.  **`FE_Validation` (ForEach Loop - Parallel):**
    * Loops through the same table list.
    * **`Lookup_GetSourceCount`:** Runs `SELECT COUNT_BIG(*) AS [RowCount]...` on the source table.
    * **`Lookup_GetSinkCount`:** Runs `SELECT COUNT_BIG(*) AS [RowCount]...` on the sink table.
    * **`If_CountsMatch` (If Activity):**
        * **True:** Does nothing.
        * **False:** Triggers a `Fail` activity, stopping the pipeline and reporting the exact table and mismatched counts.
5.  **`Script_EnableConstraints` (Script Activity):**
    * If all validations pass, this final script runs dynamic SQL to re-enable all foreign key constraints.

### Pipeline 2: `PL_Incremental_Sync_CDC` (Next Phase)
This pipeline (not part of the initial load) runs on a schedule (e.g., every 5 minutes) to move changes.

1.  **`Mapping Data Flow`:**
    * **Source:** Connects to the on-prem SQL Server and enables "Change Data Capture (native)".
    * **Sink:** Connects to Azure SQL DB and uses "Update method" (`Allow insert`, `Allow upsert`, `Allow delete`) to apply the changes.

---

## 4. Setup & Installation

### A. On-Premise SQL Server (Source)
1.  Restore the `AdventureWorks` backup to your local SQL Server instance.
2.  **Install the Self-Hosted Integration Runtime (SHIR):** Install the SHIR on the same machine (or a machine with network access to it). Connect it to your ADF instance.
3.  **Ensure SQL Server Agent is running:** This is required for CDC.
4.  **Enable CDC:** Run the following scripts on the `AdventureWorks` database:
    ```sql
    -- 1. Enable on the Database
    EXEC sys.sp_cdc_enable_db;
    GO
    
    -- 2. Enable on all tables (using the automation script)
    -- (This script loops and enables CDC on all user tables)
    DECLARE @sql NVARCHAR(MAX) = N'';
    SELECT @sql += N'EXEC sys.sp_cdc_enable_table @source_schema = N''' + s.name + N''', @source_name = N''' + t.name + N''', @role_name = NULL;' + CHAR(13)
    FROM sys.tables t JOIN sys.schemas s ON t.schema_id = s.schema_id
    WHERE t.is_ms_shipped = 0 AND t.is_tracked_by_cdc = 0;
    EXEC sp_executesql @sql;
    GO
    ```

### B. Azure (Target)
1.  **Create Azure SQL Database:** Provision a new Azure SQL Database (e.g., `AdventureWorks`).
2.  **Run Schema Script:** Run the `schema.sql` script (generated from SSMS Schema Compare) against the empty Azure SQL DB to create all tables, views, procedures, etc.
3.  **Create Full-Text Index:** Manually run the following to fix the dependency for `uspSearchCandidateResumes`:
    ```sql
    CREATE FULLTEXT INDEX ON [HumanResources].[JobCandidate]([Resume] LANGUAGE 1033)
    KEY INDEX PK_JobCandidate_JobCandidateID 
    ON [AW2016FullTextCatalog];
    GO
    ```
4.  **Firewall:** In the Azure SQL Server's "Networking" settings, ensure **"Allow Azure services and resources to access this server"** is **checked**.

### C. Azure Data Factory (ADF)
1.  **Linked Services:**
    * `LS_OnPrem_SQL`: Connects to your on-prem SQL Server via the **SHIR**.
    * `LS_Azure_SQL`: Connects to your Azure SQL Database.
2.  **Datasets:**
    * `DS_Param_OnPremSQL_Table`: Parameterized dataset for SQL Server.
        * Parameters: `SchemaName` (String), `TableName` (String)
        * Connection: Set schema to `@dataset().SchemaName` and table to `@dataset().TableName`.
    * `DS_Param_AzureSQL_Table`: Parameterized dataset for Azure SQL DB.
        * Parameters: `SchemaName` (String), `TableName` (String)
        * Connection: Set schema to `@dataset().SchemaName` and table to `@dataset().TableName`.
3.  **Pipelines:**
    * Build the `PL_Initial_Load_Dynamic` pipeline as described in the architecture section.
    * Ensure all dynamic content expressions are set correctly.

---

## 5. How to Run the Migration
1.  **Publish:** Save all changes in ADF and **Publish** them.
2.  **Trigger:** Manually trigger the `PL_Initial_Load_Dynamic` pipeline.
3.  **Monitor:** Go to the "Monitor" tab. The pipeline will run for several minutes.
    * If it succeeds, all data is copied and validated.
    * If it fails, check the error message. It will tell you exactly which table failed validation or which activity had a problem.

---

## 6. Validation Checks Performed
This pipeline validates the migration by:
* **Row Count Validation:** Ensures `COUNT(*)` on the source table matches `COUNT(*)` on the sink table for all tables.
* **Foreign Key Validation:** The final `Script_EnableConstraints` activity implicitly validates all foreign key relationships. If bad data was inserted (e.g., an `OrderID` with no `CustomerID`), this step would fail, alerting you to a data integrity issue.
* **Computed Column Validation:** Validated by design. By not copying computed columns, we ensure the data is recalculated at the destination, guaranteeing the formula is correctly applied.
