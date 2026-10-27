# Azure Data Factory Lakehouse Pipeline: Airline Bookings

An end-to-end ETL/ELT project built with **Azure Data Factory (ADF)** and **ADLS Gen2**, following the **Medallion architecture** (Bronze → Silver). Raw data from an API, an on-premises file and a SQL database is ingested into a data lake, then cleaned, joined, de-duplicated and loaded into a **Delta table**.

## Tech Stack

- **Azure Data Factory:** pipelines, mapping data flows, triggers
- **Azure Data Lake Storage Gen2:** Bronze and Silver storage
- **Azure SQL Database:** relational source **[CHECK]**
- **Delta Lake:** Silver table with upsert (merge) support
- **GitHub:** source control integration (collaboration branch `main`, publish branch `adf_publish`)

## Architecture

```
 REST API ─────────┐
 On-prem file ─────┼──► Bronze (ADLS Gen2, raw) ──► Data Flow ──► Silver (Delta table)
 Azure SQL / SQL ──┘                                (clean, join,
                                                     de-duplicate)
```

| Layer | What it holds | Built by |
|---|---|---|
| **Bronze** | Raw data exactly as received | `API_Ingestion`, `onprem_ingestion`, `sqltoazuredatalake` |
| **Silver** | Cleaned, joined, de-duplicated data in Delta format | `silverlayer` + `data Transformation` data flow |

## Pipelines

| Pipeline | Purpose |
|---|---|
| `API_Ingestion` | Pulls data from a REST API and lands it in the data lake (datasets `ds_Api_source`, `ds_api_sink`) |
| `onprem_ingestion` | Copies a file from an on-premises source into the data lake (dataset `ds_onprem_file`) **[CHECK: self-hosted integration runtime?]** |
| `sqltoazuredatalake` | Copies tables from SQL into ADLS Gen2 |
| `parentpipeline` | Orchestrates the ingestion pipelines with Execute Pipeline activities **[CHECK]** |
| `silverlayer` | Runs the `data Transformation` data flow (Bronze → Silver) |
| `psudopipeline` | Test/practice pipeline **[CHECK]** |

## Data Flow: `data Transformation`

**Sources**
- `FactBookings`: booking transactions
- `DimAirline`, `DimFlight`, `DimPassenger`: dimension tables (also airport data **[CHECK]**)

**Transformations**
1. **Select**: renames columns to a clean schema (for example `passenger_id`, `full_name`, `gender_flag`, `age`, `country`)
2. **Derived Column**: derives the `country` column and fixes data types (for example `toInteger(age)`)
3. **Join (`Airlinejoin`)**: joins bookings to the airline dimension, with matching key types
4. **Aggregate (`aggregate1`)**: groups by `booking_id` and keeps the first row of every other column, using a column pattern with `first($$)`
5. **Sink (`sink4`)**: **upserts** into a Delta table, using `booking_id` as the key

## Problems Solved

| Problem | Cause | Fix |
|---|---|---|
| Garbled column names (`PAR1...`) | A Parquet file was read through a CSV (delimited text) dataset | Created a Parquet dataset and re-imported the projection |
| `string` vs `integer` type mismatch errors | Columns were typed as string after a wrong dataset type | Used `toInteger()` / `toString()` and fixed types at the source |
| Delta merge failure: "multiple source rows matched" | 1000 rows but only 885 distinct `booking_id` values | Added an Aggregate before the sink to de-duplicate on `booking_id` |
| Join error: column not present in context | Ambiguous column names between streams | Used `StreamName@column` references and computed columns |

## Repository Structure

```
├── pipeline/        # ADF pipelines (JSON)
├── dataset/         # Source and sink datasets
├── dataflow/        # Mapping data flows
├── linkedService/   # Connections (no secrets stored)
├── trigger/         # Schedule triggers [CHECK]
└── README.md
```

## How to Run

1. Create an Azure Data Factory and connect it to this repository (**Manage → Git configuration**).
2. Create linked services for ADLS Gen2, Azure SQL and the API source, and add your own credentials. No secrets are stored in this repo.
3. Turn on **Data flow debug**, then run `parentpipeline`, followed by `silverlayer`.
4. Check the Delta table in the Silver container.

## Key Learnings

- The dataset format must match the actual file format (CSV vs Parquet).
- Delta upserts need **unique keys** in the incoming data, so always de-duplicate before a merge sink.
- Join keys must have the same data type on both sides.
- Medallion layering keeps raw data untouched while Silver stays clean and queryable.

## Author

**Vijay K**: Azure Data Engineering | SQL | Python | Power BI
[LinkedIn](https://linkedin.com/in/vijay-k) · [GitHub](https://github.com/Vijayhanu)
