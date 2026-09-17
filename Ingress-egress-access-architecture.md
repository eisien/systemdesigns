# Azure Databricks Data Platform: Ingestion, Egress & Access Architecture

---

## 1. Data Ingestion

### Pattern A — Batch / Scheduled via ADF

**Best for:** ERP systems (SAP), relational databases, file drops, SaaS APIs.

```
Source System
  → ADF Pipeline (Copy Activity / Data Flow)
    → ADLS Gen2 (landing/raw zone)
      → Databricks Auto Loader (readStream + cloudFiles)
        → Delta Lake (bronze → silver → gold)
```

- ADF owns orchestration, scheduling, retry logic, and secret management via Key Vault linked services.
- Use ADF **Managed Identity** (not SPN with secrets) for ADLS access — assign `Storage Blob Data Contributor` on the landing container only.
- ADF writes raw files to a dedicated **landing container** — Databricks picks it up from there. Never write directly into Delta tables from ADF.

### Pattern B — Auto Loader (Streaming Ingest from ADLS)

**Best for:** high-frequency file drops, event-driven pipelines.

```python
spark.readStream.format("cloudFiles") \
    .option("cloudFiles.format", "parquet") \
    .option("cloudFiles.schemaLocation", "/checkpoints/schema") \
    .load("abfss://landing@<storage>.dfs.core.windows.net/source/")
```

- Handles schema evolution, new file detection, and exactly-once semantics automatically.
- Preferred over raw Structured Streaming for ADLS-sourced file ingestion.

### Pattern C — Event-Driven Streaming (Kafka / Event Hubs)

**Best for:** real-time telemetry, IoT, change data capture (CDC).

```
Source
  → Event Hubs (Kafka-protocol endpoint)
    → Databricks Structured Streaming
      → Delta Lake (append / merge)
```

- Use **Event Hubs with Kafka protocol** — no connector changes needed, native Kafka API.
- For CDC: Debezium → Event Hubs → Databricks, or Azure DB → DMS → ADLS → Auto Loader.

### Pattern D — Direct JDBC Ingestion

**Best for:** one-time historical loads, small operational tables.

```python
df = spark.read.format("jdbc") \
    .option("url", "jdbc:sqlserver://...") \
    .option("dbtable", "schema.table") \
    .option("user", dbutils.secrets.get("scope", "db-user")) \
    .load()
```

- Always use **Databricks Secrets** backed by Azure Key Vault — never inline credentials.
- For large tables, partition by a numeric or date column to parallelise reads.

---

## 2. Storage Layout — ADLS Gen2

```
storage account
├── landing/    ← ADF writes here (raw files, transient)
├── bronze/     ← Auto Loader ingests raw Delta tables
├── silver/     ← Cleansed / conformed Delta tables
├── gold/       ← Aggregated business-level data products
└── sandbox/    ← Dev/exploration (separate container, no prod data)
```

- Each zone is a **separate container** — independent RBAC, lifecycle policies, and soft-delete settings.
- Unity Catalog **external locations** map Databricks to these containers via a storage credential (managed identity).
- Disable public network access on ADLS — only the Databricks managed identity gets access.

---

## 3. Access Control — Unity Catalog + Entra ID

### Governance Stack

```
Entra ID (identities)
  → Databricks Account Groups (synced via SCIM)
    → Unity Catalog GRANTs (catalogs, schemas, tables)
      → SQL Warehouse (compute — enforces UC permissions at query time)
```

### Unity Catalog Object Hierarchy

```
Metastore  (one per region, account-level)
├── Catalog  (one per domain / product team)
│   ├── Schema  (logical grouping / subject area)
│   │   ├── Table / View / Function
│   │   └── Volume  (unstructured files)
│   └── External Location → ADLS container
└── Storage Credential  (managed identity → ADLS)
```

### Entra Group Ownership

The principle: **the team accountable for a capability owns the group**, not the platform team.
The platform team owns only infrastructure-level groups.

| Entra Group | Owned By | Databricks Role |
|---|---|---|
| `grp-databricks-account-admins` | Platform / Infra team | Account Admin |
| `grp-databricks-workspace-admins-<env>` | Platform / Infra team | Workspace Admin |
| `grp-adf-pipeline-mi` | Platform / Infra team | Storage Blob Contributor on landing container |
| `grp-data-engineers-<domain>` | Domain engineering lead | Catalog/schema owner, CREATE TABLE, MODIFY |
| `grp-data-consumers-<domain>` | **Data product owner** | SELECT on published gold tables |
| `grp-data-stewards-<domain>` | Business / governance lead | BROWSE + lineage access in Catalog Explorer |

> **On BI service principals:** the consuming team (BI, Finance, external app) registers and manages their own service principal in Entra — they own the *identity*. But the data product owner controls *access*: they decide which service principals get added to `grp-data-consumers-<domain>`, and therefore which tools can read their published data. Separating identity management from access grants preserves the accountability chain.

### GRANT Pattern

```sql
-- Domain engineers can build within their catalog
GRANT USE CATALOG, USE SCHEMA, CREATE TABLE, MODIFY
  ON CATALOG production_mi TO `grp-data-engineers-sop`;

-- Consumers (BI tools, apps, other teams) can only read published gold
GRANT USE CATALOG, USE SCHEMA, SELECT
  ON SCHEMA production_mi.gold TO `grp-data-consumers-finance`;
```

---

## 4. Data Egress — Exposing Data Products

### Access Protocol Decision Guide

| Scenario | Protocol | Auth |
|---|---|---|
| BI tools (Power BI, Tableau) on-corp | JDBC/ODBC → SQL Warehouse | OAuth 2.0 service principal |
| Power BI large semantic models | Direct Lake (reads Delta from ADLS directly) | OAuth 2.0 service principal |
| Application / API querying data | SQL Statement Execution REST API | OAuth 2.0 M2M |
| External partner / cross-org | Delta Sharing (OpenSharing) | Sharing bearer token |
| Internal separate Databricks workspace | Databricks-to-Databricks sharing | Unity Catalog federation |
| Downstream pipeline needing files | ADLS export (Parquet/Delta) via scheduled job | Managed identity |
| Real-time streaming consumer | Event Hubs subscription | Event Hubs SAS / Entra |

---

### Pattern A — JDBC/ODBC (BI Tools)

```
Power BI / Tableau / Excel
  → Databricks JDBC Driver
    → SQL Warehouse (serverless preferred)
      → Unity Catalog (row/column-level security enforced here)
```

**Connection details:**
```
Server:    <workspace-id>.azuredatabricks.net
HTTP Path: /sql/1.0/warehouses/<warehouse-id>
Auth:      OAuth 2.0 (Entra service principal)
```

- Never use PATs for production BI connections — use OAuth 2.0 with an Entra service principal.
- The service principal must be a member of `grp-data-consumers-<domain>` — access is granted and revoked by the data product owner.
- Power BI **Direct Lake mode** reads Delta files from ADLS without hitting the SQL Warehouse — fastest path, lowest cost.

---

### Pattern B — REST API (Application / Platform Consumer)

```
External App / Platform (e.g. Workiva)
  → Databricks Statement Execution API
    → SQL Warehouse
      → Unity Catalog
```

```bash
curl -X POST https://<workspace>.azuredatabricks.net/api/2.0/sql/statements \
  -H "Authorization: Bearer <oauth-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "warehouse_id": "<id>",
    "statement": "SELECT * FROM production_mi.gold.production_volume LIMIT 1000"
  }'
```

- Use OAuth 2.0 machine-to-machine (M2M) flow — register an Entra app, grant it `CAN USE` on the warehouse, and add its service principal to the relevant consumer group.
- Results are paginated; poll the returned `statement_id` for large result sets.

---

### Pattern C — Delta Sharing / OpenSharing (External / Partner Access)

**Best for:** cross-organisation sharing, partners, or tools that cannot connect via JDBC.

```
Provider (your Databricks workspace)
  → Share (logical collection of Delta tables)
    → Recipient (external org, identified by sharing token)
      → Consumer reads via Delta Sharing client (Python, Spark, Power BI connector, pandas)
```

```sql
-- Provider: create and publish
CREATE SHARE finance_data_product;
ALTER SHARE finance_data_product ADD TABLE production_mi.gold.production_volume;

-- Grant access to a named recipient
CREATE RECIPIENT workiva_integration COMMENT 'Workiva platform access';
GRANT SELECT ON SHARE finance_data_product TO RECIPIENT workiva_integration;
```

- Recipients authenticate with a **bearer token** — auto-expiring and revocable.
- Data is streamed on demand via your platform's Delta Sharing REST endpoint — data never leaves your ADLS.
- Full audit trail logged in Unity Catalog system tables (`system.access.audit`).
- Ownership: the data product owner creates the Share and controls which Recipients can access it.

---

### Pattern D — Databricks-to-Databricks (Internal Cross-Workspace)

For internal teams on separate workspaces or accounts (e.g. Finance team on their own workspace):

```sql
-- Provider workspace
CREATE SHARE sop_gold_share;
ALTER SHARE sop_gold_share ADD TABLE production_mi.gold.production_volume;
GRANT SELECT ON SHARE sop_gold_share TO RECIPIENT finance_workspace_recipient;

-- Consumer workspace — mounts as a read-only catalog
CREATE CATALOG finance_sop_data USING SHARE <provider-metastore-id>.sop_gold_share;
-- Consumer queries it like any local table
SELECT * FROM finance_sop_data.gold.production_volume;
```

- Unity Catalog handles authentication transparently between workspaces.
- The consumer workspace has read-only access — they cannot modify the source data.

---

## 5. Network Security

```
External / On-prem
  ↓
Azure Private Endpoint  (Databricks + ADLS + Key Vault + Event Hubs)
  ↓
VNet-injected Databricks Workspace
  ↓
Private Link → ADLS Gen2  (public endpoint disabled)
  ↓
Unity Catalog Metastore  (account-level, separate plane)
```

- Enable **Private Endpoints** for all platform services: Databricks workspace, ADLS, Key Vault, Event Hubs.
- ADLS public network access disabled — only Databricks managed identity and ADF managed identity have access.
- SQL Warehouse JDBC/ODBC can remain on public internet over TLS — authentication (OAuth 2.0) and authorisation (Unity Catalog GRANTs) are the security layer. Add IP allowlists if stricter network control is required.
- For on-prem sources via ADF: use **Self-Hosted Integration Runtime** in the corporate network, communicating via TLS over internet or ExpressRoute.

---

## 6. Ownership Summary

```
Platform / Infra Team
├── Databricks Account (metastore, workspace provisioning)
├── ADLS storage accounts and container ACLs
├── ADF infrastructure (not individual pipelines)
├── Azure Key Vault (platform secrets)
├── Network: private endpoints, VNet, NSGs
└── Entra groups: account-admins, workspace-admins, adf-mi

Domain Data Engineering Team (per domain)
├── Databricks catalogs and schemas for their domain
├── Delta tables, views, functions in their catalog
├── ADF pipelines for their domain's ingestion
├── Job clusters / compute policies for their workloads
└── Entra groups: data-engineers-<domain>

Data Product Owner (per product)
├── Declares who can consume (grants to consumer groups)
├── Controls membership of grp-data-consumers-<domain>
├── Creates and manages Shares for external/cross-workspace access
├── Owns data quality SLAs and documentation in Unity Catalog
└── Entra groups: data-consumers-<domain>, data-stewards-<domain>

Consuming Team (BI, App, External Partner)
├── Registers and manages their own service principal in Entra
├── Requests access through the data product owner, Data product owner adds that service principal to their consumer group
└── Owns nothing in the provider workspace — access only
```

---

## 7. References

- [Connect to a SQL warehouse — Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/compute/sql-warehouse/)
- [Share data and AI assets securely — Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/data-sharing/)
- [Connect From Anywhere to Databricks SQL](https://www.databricks.com/blog/2022/06/29/connect-from-anywhere-to-databricks-sql.html)
- [A Practical Guide to Catalog Layout, Data Sharing and Distribution with Databricks Unity Catalog](https://medium.com/databricks-unity-catalog-sme/a-practical-guide-to-catalog-layout-data-sharing-and-distribution-with-databricks-unity-catalog-f34fa822a367)
- [Access Control and Networking Security with Power BI and Databricks](https://community.databricks.com/t5/technical-blog/access-control-and-networking-security-with-power-bi-and/ba-p/66779)
