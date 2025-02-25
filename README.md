# End-to-End Azure Data Engineering Pipeline Setup

## Overview
This guide provides a step-by-step process for building an Azure Data Engineering Pipeline that ingests on-premises SQL Server data, processes it through Bronze, Silver, and Gold layers, and visualizes it in Power BI. The pipeline follows a Medallion Architecture and incorporates Azure Databricks, Azure Synapse Analytics, Azure Data Factory (ADF), Azure Active Directory (AAD), and Azure Key Vault for security and governance.

## 1. Environment Setup
### 1.1 Azure Resource Setup
#### Create a Resource Group (RG)
- Go to Azure Portal → Search Resource Groups → Create a new RG.

#### Create Azure Data Factory (ADF)
- Go to Azure Portal → Create a new Data Factory → Choose Azure Integration Runtime.

#### Create Storage Account & Containers (Bronze, Silver, Gold)
- Go to Azure Storage Accounts → Create New Storage Account.
- Inside the Data Lake (ADLS Gen2), create three containers:
  - Bronze (raw data)
  - Silver (cleaned and structured data)
  - Gold (final transformed data for reporting)

#### Create Azure Databricks Workspace
- Go to Azure Databricks → Create a workspace.
- This will generate a Databricks Resource Group automatically.

#### Create Azure Synapse Analytics Workspace
- Go to Azure Synapse Analytics → Create New Workspace.
- Ensure Data Lake Storage (Gen2) is linked.

#### Set Up Azure Key Vault (For Secure Credentials Storage)
- Go to Azure Key Vault → Create a new Key Vault.
- Add secrets (SQL username/password, Databricks Token, etc.).

## 2. Setting Up SQL Server On-Premises
### 2.1 Install & Configure SQL Server
- Download SQL Server and SQL Server Management Studio (SSMS).
- Download AdventureWorksLT database and restore it in SSMS:
  - Move `.bak` file to `C:\Program Files\Microsoft SQL Server\MSSQL16.SQLEXPRESS\MSSQL\Backup`.
  - Use SSMS to restore the database.

### 2.2 Configure Authentication & Permissions
- Create SQL login:
```sql
CREATE LOGIN mrk WITH PASSWORD = 'YourStrongPassword';
```
- Grant permissions:
```sql
USE AdventureWorksLT2019;
GRANT SELECT ON SCHEMA::SalesLT TO mrk;
```

### 2.3 Store SQL Credentials in Azure Key Vault
- Add username and password as secrets in Key Vault.

## 3. Data Ingestion with Azure Data Factory (ADF)
### 3.1 Install Self-Hosted Integration Runtime
- In ADF → Manage → Integration Runtimes → Create Self-Hosted IR.
- Download and install the runtime on the on-prem SQL Server.

### 3.2 Create a Pipeline to Copy Data from On-Prem SQL Server to Bronze
#### Create a new pipeline in ADF
- Add **Copy Data Activity**:
  - **Source**:
    - Create a new linked service → Choose SQL Server.
    - Use Key Vault for authentication.
  - **Sink**:
    - Create a Parquet dataset.
    - Destination path: `/mnt/bronze/SalesLT/TableName/`.

#### Use Lookup & Foreach Activity to Copy All Tables
Query for all tables:
```sql
SELECT s.name AS SchemaName, t.name AS TableName
FROM sys.tables t
INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
WHERE s.name = 'SalesLT';
```
- In ADF, Lookup Activity → Foreach Activity → Copy Data for each table.
- Run & Monitor Pipeline Execution → **Debug and Trigger Now**.

## 4. Data Transformation with Azure Databricks
### 4.1 Mount ADLS in Databricks
- Create a Databricks cluster.
- Mount Storage:
```python
configs = {"fs.azure.account.auth.type": "CustomAccessToken",
           "fs.azure.account.custom.token.provider.class":
           spark.conf.get("spark.databricks.passthrough.adls.gen2.tokenProviderClassName")}

dbutils.fs.mount(
    source="abfss://bronze@yourstorageaccount.dfs.core.windows.net/",
    mount_point="/mnt/bronze",
    extra_configs=configs)
```

### 4.2 Bronze to Silver (L1 Transformation)
- Read from Bronze, clean the data, and store it in Silver.
```python
df = spark.read.format("delta").load("/mnt/bronze/SalesLT/Address")
df = df.withColumn("ModifiedDate", df["ModifiedDate"].cast("date"))
df.write.format("delta").mode("overwrite").save("/mnt/silver/SalesLT/Address")
```

### 4.3 Silver to Gold (L2 Transformation)
- Standardize column names and aggregate data.
```python
df = spark.read.format("delta").load("/mnt/silver/SalesLT/Address")
df = df.withColumnRenamed("CustomerID", "CUSTOMER_ID")
df.write.format("delta").mode("overwrite").save("/mnt/gold/SalesLT/Address")
```

### 4.4 Automate with ADF Pipelines
- Create a new ADF pipeline.
- Add **Databricks Notebook Activity** for:
  - Bronze to Silver
  - Silver to Gold
- Trigger on Successful Completion.

## 5. Load Data into Synapse Analytics
### 5.1 Create External Views in Synapse
- In Synapse Studio, create a serverless SQL database (`gold_db`).
- Create a stored procedure to dynamically create views:
```sql
CREATE OR ALTER PROC CreateSQLServerlessView_gold @viewName NVARCHAR(100)
AS
BEGIN
    DECLARE @statement NVARCHAR(MAX);
    SET @statement = N'CREATE OR ALTER VIEW ' + QUOTENAME(@viewName) + N' AS
        SELECT * FROM OPENROWSET(
            BULK ''https://yourstorageaccount.dfs.core.windows.net/gold/SalesLT/' + @viewName + N'/'' ,
            FORMAT = ''DELTA'') AS result;';
    EXEC sp_executesql @statement;
END;
```
- Automate View Creation using Pipeline in Synapse.

## 6. Data Visualization with Power BI
### 6.1 Connect Power BI to Synapse
- **Get Data** → Azure Synapse Analytics → Enter Serverless SQL Endpoint.
- Select Tables and Load.

### 6.2 Create Visuals
- Use fact and dimension tables to build dashboards.
- Add filters and slicers for user interaction.

## 7. Security & Governance with Azure AD & Key Vault
### 7.1 Role-Based Access Control (RBAC) using Azure AD
- Create Azure AD Security Groups for Data Engineers, Analysts, Admins.
- Assign roles in IAM settings (e.g., Storage Blob Data Contributor).

### 7.2 Secure Credentials with Key Vault
- Store SQL Credentials, Storage Access Keys, Databricks Tokens.
- Assign Managed Identities to ADF, Synapse, Databricks.

## 8. Automate Pipeline Execution
### 8.1 Create a Scheduled Trigger in ADF
- Set up a **daily trigger** to run the pipeline.

### 8.2 Validate End-to-End Pipeline
- Insert a new row in SQL Server and check:
  - Data processed in Bronze, Silver, Gold.
  - Power BI dashboard updates automatically.

