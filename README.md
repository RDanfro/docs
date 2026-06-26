# Branch CX Intelligence Pipeline

An end-to-end SQL Server BI solution consolidating customer experience, complaints, and financial performance across a simulated 10-branch banking network into one automated reporting pipeline.

Built as an independent project to extend existing Power BI/SQL skills into the SQL Server BI ecosystem — T-SQL, SSIS, SSRS.

## Business problem

Monthly CX reporting was fragmented across manual spreadsheets, with no consolidated view linking complaints, satisfaction scores (NPS/CES), and branch financial performance — making it hard to separate genuine service-quality issues from simple high transaction volume.

## What this delivers

| Capability | Tool |
|---|---|
| Normalised relational schema for CX/branch data | SQL Server |
| Realistic simulated dataset (10 branches × 12 months) | T-SQL |
| NPS, CES trend, and complaint-ranking analytics | T-SQL — CTEs & window functions |
| Automated, parameterised monthly KPI calculation | Stored procedure w/ error handling |
| ETL pipeline design for nightly ingestion | SSIS (Control Flow + Data Flow) |
| Governed management report with RAG status & regional rollups | SSRS (Report Builder) |

## Architecture

```
Source data → SSIS ETL (Extract/Validate/Load) → SQL Server (5 tables)
→ Stored Procedure (monthly KPI calc) → Staging_BranchKPI → SSRS Report
```
Diagrams: [`/docs/SSIS_ControlFlow.png`](./docs/SSIS_ControlFlow.png), [`/docs/SSIS_DataFlow.png`](./docs/SSIS_DataFlow.png)

## Database schema

| Table | Purpose |
|---|---|
| `dbo.Branches` | Dimension — 10 branches, 4 regions, REM owners |
| `dbo.CustomerComplaints` | Fact — complaints by category, channel, resolution status |
| `dbo.PulseSurveys` | Fact — NPS (1–10) and CES (1.0–5.0) responses |
| `dbo.BranchDeposits` | Fact — monthly deposits by branch/product |
| `dbo.Staging_BranchKPI` | Pre-aggregated output, populated by the stored procedure |

Indexed on branch + date for query performance. Schema definition included in [`/docs/SQL_script.txt`](./docs/SQL_script.txt)

## Simulated dataset

Confidential real data was replaced with a T-SQL-generated dataset with intentional patterns: two branches (`ACC-002`, `TEM-001`) weighted toward higher complaints; branch-level bias on NPS/CES so results are traceable, not random. ~650–1,450 complaints, 1,200 survey responses, 120 deposit records across 12 months. Generation scripts included in [`/docs/SQL_script.txt`](./docs/SQL_script.txt)

## Analytical queries

- **Branch NPS** — promoters/detractors via chained CTEs, `NULLIF`-guarded against divide-by-zero
- **Top complaint categories** — `ROW_NUMBER()` per branch, filtered to top 3
- **CES trend detection** — `LAG()` per branch, classified Improving/Declining/Stable
- **Regional complaint ranking** — `DENSE_RANK()` + `SUM() OVER()` for rank and % of regional volume

Queries included in [`/docs/SQL_script.txt`](./docs/SQL_script.txt)

## Stored procedure

`usp_GenerateBranchKPIReport(@ReportMonth)` — joins complaints, surveys, and deposits into per-branch monthly KPIs; wrapped in `TRY/CATCH` logging to `dbo.SSIS_ErrorLog` so failures are captured, not silent.

```sql
EXEC dbo.usp_GenerateBranchKPIReport @ReportMonth = '2024-12-01', @RowsAffected = @rows OUTPUT;
```
Script included in [`/docs/SQL_script.txt`](./docs/SQL_script.txt)

## SSIS pipeline design

**Control Flow:** truncate staging → Data Flow load → execute stored procedure → success email, with `OnError` handler routing to a failure alert.
**Data Flow:** flat file source → data conversion → conditional split, rejected rows redirected to the error log rather than dropped.

Diagrams: [`/docs/SSIS_ControlFlow.png`](./docs/SSIS_ControlFlow.png), [`/docs/SSIS_DataFlow.png`](./docs/SSIS_DataFlow.png)

## SSRS management report

Parameterised branch scorecard (`@ReportMonth`) with RAG colour formatting on CES/complaints, regional grouping with correct footer aggregation (`Sum()` for counts, `Avg()` for CES/NPS — avoiding misleading inflated subtotals), and a footer with generation timestamp and page number (VB.NET expressions).

Files: [`/docs/BranchScorecard.rdl`](./docs/BranchScorecard.rdl), [`/docs/BranchScorecard_Dec2024.pdf`](./docs/BranchScorecard_Dec2024.pdf), [`/docs/BranchScorecard_Screenshot.png`](./docs/BranchScorecard_Screenshot.png)

## How to run

Requires SQL Server Express, SSMS, and Report Builder (all free).

1. Open [`/docs/SQL_script.txt`](./docs/SQL_script.txt) — it contains the schema, data generation, analytical queries, and stored procedure in sequence
2. Run the **schema section** first to create the database/tables
3. Run the **data generation section** to populate the database
4. Run the **stored procedure section** to create `usp_GenerateBranchKPIReport`, then execute it per month
5. Run the **CTE / window function query sections** in SSMS to explore results
6. Open `BranchScorecard.rdl` in Report Builder, point at your local DB, click **Run**

## Repository structure

```
branch-cx-intelligence-pipeline/
├── README.md
└── docs/
    ├── SQL_script.txt
    ├── SSIS_ControlFlow.png
    ├── SSIS_DataFlow.png
    ├── BranchScorecard.rdl
    ├── BranchScorecard_Dec2024.pdf
    └── BranchScorecard_Screenshot.png
```
## About

**Richard Danfro** — Customer Insights & Analytics, Consolidated Bank Ghana. 10+ years in CX measurement, survey research, and feedback automation; built independently to extend Power BI/SQL/Python skills into SQL Server BI.
[LinkedIn](https://www.linkedin.com/in/richard-danfro/)
