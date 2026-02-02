# Excel to BigQuery Ingestion Framework – Full Documentation

---

## 1. Overview

The **Excel to BigQuery Ingestion Framework** is a configurable, metadata-driven pipeline designed to ingest Excel files from Google Cloud Storage (GCS), apply flexible parsing and transformation rules, and load the data into BigQuery in a layered architecture.

This framework supports:

* Complex Excel layouts (missing headers, multi-row headers, footers)
* Schema evolution
* Multi-sheet processing
* Land → Ingest layer separation
* Auditing and lineage
* Idempotent reprocessing
* Operational resilience and traceability

It is designed for enterprise-grade ingestion workflows and aligns with modern data lakehouse best practices.

---

## 2. Architecture

### 2.1 High-Level Flow

```
GCS (Excel Files)
        ↓
File Selector & Job Orchestration
        ↓
Excel Parser (Sheet-Level Rules Applied)
        ↓
Land Layer (Raw, Auditable Load)
        ↓
Optional Transformations / Validation
        ↓
Ingest Layer (Analytics-Ready Tables)
        ↓
Post-Processing (Archive/Delete/None)
```

### 2.2 Layered Data Model

| Layer  | Purpose            | Characteristics                                       |
| ------ | ------------------ | ----------------------------------------------------- |
| Land   | Raw ingestion      | Schema evolution allowed, append-only, audit-friendly |
| Ingest | Cleaned/structured | Deduplicated, validated, analytics-ready              |

---

## 3. Configuration Philosophy

The framework is **YAML-driven**, using a hierarchical override model:

1. **Sheet-Level Rules** – Highest priority
2. **Job-Level Rules** – Override global defaults
3. **Global Defaults** – Fallback configuration

This allows:

* Reuse of global standards
* Job-specific customizations
* Sheet-specific exceptions

---

## 4. Global Configuration Section

### 4.1 Project & Environment Settings

```yaml
global_config:
  gcp_project_id: "pmtech-excel-processor-sbx"
```

Defines the target Google Cloud project where all BigQuery operations will occur.

---

### 4.2 Audit Configuration

```yaml
audit:
  dataset: "excel_processor_land"
  table: "sales_audit"
```

Purpose:

* Tracks file-level processing metadata
* Enables idempotency
* Supports operational troubleshooting

Recommended audit fields:

* job_id
* file_name
* file_path
* sheet_name
* status (PROCESSING, COMPLETED, FAILED, SKIPPED)
* start_time
* end_time
* rows_read
* rows_loaded
* error_message

---

### 4.3 Loading Defaults

```yaml
loading_defaults:
  continue_on_error: false
  skip_if_exists: false
```

| Field             | Description                                           |
| ----------------- | ----------------------------------------------------- |
| continue_on_error | If true, row-level failures do not stop the job       |
| skip_if_exists    | Prevents reprocessing the same file if already loaded |

---

### 4.4 Global Parsing Rules

```yaml
parsing_rules:
  is_header_missing: false
  ordered_header: []
  group_header:
    enabled: false
    rows: [0]
    separator: "_"
    direction: "prefix"
    ignore_labels: []
  skip_footer: 0
  header_row: 0
  data_start: 1
  data_end: null
  usecols: null
  trim_whitespace: true
  treat_as_null: ["N/A", "NULL", "nan"]
  force_string_cols: []
  column_mapping: {}
```

#### Field Descriptions

| Field                      | Description                                         |
| -------------------------- | --------------------------------------------------- |
| is_header_missing          | If true, framework generates synthetic column names |
| ordered_header             | Explicit column order if headers are missing        |
| group_header.enabled       | Enables multi-row header merging                    |
| group_header.rows          | Which rows contain header fragments                 |
| group_header.separator     | Character used to join header fragments             |
| group_header.direction     | prefix or postfix when merging                      |
| group_header.ignore_labels | Skips static header labels                          |
| skip_footer                | Number of rows to ignore from bottom                |
| header_row                 | Row index containing column names                   |
| data_start                 | Row index where actual data begins                  |
| data_end                   | Row index where data ends (null = last row)         |
| usecols                    | Excel column range to read (e.g., A:M)              |
| trim_whitespace            | Removes leading/trailing spaces                     |
| treat_as_null              | Values converted to SQL NULL                        |
| force_string_cols          | Columns forced to STRING type                       |
| column_mapping             | Excel header → BigQuery column mapping              |

---

### 4.5 Schema Evolution Controls

```yaml
schema_evolution:
  allow_add_fields: true
  allow_relax_fields: true
  fail_on_type_change: false
```

| Field               | Description                          |
| ------------------- | ------------------------------------ |
| allow_add_fields    | Automatically add new columns        |
| allow_relax_fields  | Allow required → nullable changes    |
| fail_on_type_change | If true, stops job when type changes |

---

### 4.6 Deduplication Settings

```yaml
deduplication:
  enabled: false
  primary_keys: []
  strategy: "latest"
```

| Field        | Description                         |       |        |
| ------------ | ----------------------------------- | ----- | ------ |
| enabled      | Enables deduplication logic         |       |        |
| primary_keys | List of columns defining uniqueness |       |        |
| strategy     | latest                              | first | reject |

---

### 4.7 Metadata Column Injection

```yaml
add_metadata_columns:
  file_name: true
  file_path: true
  load_timestamp: true
  job_id: true
```

Adds operational lineage columns into land/ingest tables.

---

### 4.8 Destination Defaults

```yaml
destination:
  type: "bigquery"
  layers:
    land:
      dataset: "excel_processor_land"
    ingest:
      dataset: "excel_processor_ingest"
```

Defines default datasets for land and ingest layers.

---

## 5. Job Configuration Section

Each job defines one ingestion workflow.

```yaml
jobs:
  - job_id: "XLSX_PROCESS_JOB"
    description: "<BRIEF_DESCRIPTION_OF_THE_PIPELINE>"
```

### 5.1 GCS Input

```yaml
gcs_input:
  bucket: "pmtech-sbx-excel-processor"
```

Defines the GCS bucket to scan for files.

---

### 5.2 File Selection Rules

```yaml
file_selector:
  match_type: "name"
  prefix: ""
  supported_formats: ["xlsx"]
  pattern: "incoming/finance_data/Financial Sample_01-14-2026.xlsx"
```

| Field             | Description                |
| ----------------- | -------------------------- |
| match_type        | name or regex              |
| prefix            | Root folder to scan        |
| supported_formats | Allowed file extensions    |
| pattern           | File name or regex pattern |

---

### 5.3 Post Processing

```yaml
post_processing:
  action: "archive"
  archive_path: "processed_logs/"
```

| Action  | Behavior                  |
| ------- | ------------------------- |
| archive | Move file to archive path |
| delete  | Delete file after success |
| none    | Leave file untouched      |

---

## 6. Sheet-Level Configuration

Each sheet can override job and global defaults.

```yaml
sheets:
  - sheet_identifier: 0
    identifier_type: "index"
```

| Field            | Description         |
| ---------------- | ------------------- |
| sheet_identifier | Sheet name or index |
| identifier_type  | name or index       |

---

### 6.1 Sheet Parsing Rules

```yaml
parsing_rules:
  is_header_missing: false
  ordered_header: []
  group_header:
    enabled: false
    rows: [0]
    separator: "_"
    direction: "prefix"
    ignore_labels: []
  skip_footer: 0
  header_row: 0
  data_start: 1
  data_end: null
  usecols: "A:M"
  trim_whitespace: true
  treat_as_null: ["N/A", "NULL", "nan"]
  force_string_cols: []
  column_mapping: {}
```

All fields behave the same as global parsing rules but override them for this sheet only.

---

### 6.2 Sheet Destination Configuration

```yaml
destination:
  type: "bigquery"
  layers:
    land:
      dataset: "excel_processor_land"
      table: "finance_data_land"
      write_mode: "WRITE_APPEND"
      schema:
        mode: "auto_detect"
        fields: []
    ingest:
      dataset: "excel_processor_ingest"
      table: "finance_data_ingest"
```

#### Land Layer Options

| Field         | Description                 |                |
| ------------- | --------------------------- | -------------- |
| dataset       | BigQuery dataset            |                |
| table         | BigQuery table              |                |
| write_mode    | WRITE_APPEND                | WRITE_TRUNCATE |
| schema.mode   | auto_detect                 | manual         |
| schema.fields | Explicit schema definitions |                |

#### Ingest Layer Options

Same as land but typically used for cleaned/curated data.

---

## 7. Processing Logic (Framework Behavior)

### 7.1 File Discovery

1. Scan GCS bucket using prefix
2. Match files based on name or regex
3. Filter by supported formats
4. Cross-check audit table to avoid reprocessing (if skip_if_exists = true)

---

### 7.2 Excel Parsing

For each file and sheet:

1. Load sheet into memory
2. Apply header logic:

   * Missing header → generate ordered_header
   * Grouped header → merge multi-row headers
3. Trim whitespace
4. Convert null values
5. Apply column mapping
6. Enforce forced STRING columns
7. Slice rows using data_start and data_end

---

### 7.3 Land Layer Load

* Write raw parsed data into land table
* Apply schema evolution rules
* Add metadata columns
* Append or truncate based on write_mode

---

### 7.4 Validation & Deduplication (Optional)

If enabled:

* Validate rows based on rules
* Deduplicate using primary_keys
* Apply conflict resolution strategy

---

### 7.5 Ingest Layer Load

* Transform and clean data
* Load into ingest table
* Enforce analytics-ready schema

---

### 7.6 Post-Processing

* Archive / Delete / No-op based on configuration
* Update audit table with final status

---

## 8. Audit & Observability

### 8.1 Audit Table Design

Recommended schema:

| Column         | Type      |
| -------------- | --------- |
| job_id         | STRING    |
| file_name      | STRING    |
| file_path      | STRING    |
| sheet_name     | STRING    |
| status         | STRING    |
| rows_read      | INTEGER   |
| rows_loaded    | INTEGER   |
| error_message  | STRING    |
| start_time     | TIMESTAMP |
| end_time       | TIMESTAMP |
| load_timestamp | TIMESTAMP |

---

### 8.2 Logging

Framework logs should include:

* Job start/end
* File discovery summary
* Sheet-level processing
* Row counts
* Schema changes
* Errors and warnings

---

## 9. Error Handling Strategy

| Scenario          | Behavior                                             |
| ----------------- | ---------------------------------------------------- |
| File not found    | Mark job as FAILED                                   |
| Sheet missing     | Log warning, skip or fail based on config            |
| Schema mismatch   | Apply evolution rules or fail                        |
| Row parsing error | Skip, quarantine, or stop based on continue_on_error |

Optional quarantine support:

```yaml
error_handling:
  on_row_error: "quarantine"
  quarantine_table: "finance_data_rejects"
```

---

## 10. Idempotency & Reprocessing

* Files tracked via audit table
* skip_if_exists prevents duplicate ingestion
* Land layer append-only
* Ingest layer can be rebuilt safely

---

## 11. Security & Access Control

* GCS access via service account
* BigQuery IAM roles:

  * roles/bigquery.dataEditor
  * roles/bigquery.jobUser
* Optional VPC Service Controls

---

## 12. Performance & Scalability

* Supports parallel sheet processing
* Partitioning and clustering supported at table level
* Schema auto-detection minimizes manual intervention
* Suitable for batch and near-real-time ingestion

---

## 13. Example End-to-End Configuration

```yaml
jobs:
  - job_id: "FINANCE_SALES_INGEST"
    description: "Monthly finance sales Excel ingestion"

    gcs_input:
      bucket: "pmtech-sbx-excel-processor"

    file_selector:
      match_type: "regex"
      prefix: "incoming/finance_data/"
      supported_formats: ["xlsx"]
      pattern: "Financial_Sample_.*\\.xlsx"

    post_processing:
      action: "archive"
      archive_path: "processed_logs/finance/"

    sheets:
      - sheet_identifier: "Sales"
        identifier_type: "name"

        parsing_rules:
          header_row: 2
          data_start: 3
          skip_footer: 1
          usecols: "A:K"
          column_mapping:
            "Invoice No": "invoice_id"
            "Customer Name": "customer_name"
            "Invoice Date": "invoice_date"
            "Amount": "invoice_amount"

        destination:
          type: "bigquery"
          layers:
            land:
              dataset: "excel_processor_land"
              table: "finance_sales_land"
              write_mode: "WRITE_APPEND"
            ingest:
              dataset: "excel_processor_ingest"
              table: "finance_sales"
```

---

## 14. Operational Best Practices

* Maintain separate configs per environment (sbx/dev/qa/prod)
* Version control YAML configs
* Use CI/CD to validate YAML syntax
* Monitor audit table regularly
* Set alerts on FAILED jobs
* Partition ingest tables by date

---

## 15. Future Enhancements (Roadmap)

* Column-level validation rules
* Row-level transformations
* PII masking and tokenization
* Change Data Capture (CDC) support
* Incremental ingestion from Excel deltas
* Metadata catalog integration

---

## 16. Appendix

### 16.1 Glossary

| Term             | Meaning                                |
| ---------------- | -------------------------------------- |
| Land Layer       | Raw ingestion zone                     |
| Ingest Layer     | Curated analytics zone                 |
| Schema Evolution | Automatic adjustment to schema changes |
| Idempotency      | Safe reprocessing without duplication  |
| Quarantine       | Storage of invalid records             |

---

### 16.2 FAQ

**Q: Can I process multiple sheets from the same file?**
Yes — define multiple entries under the `sheets` section.

**Q: Can I override dataset per job?**
Yes — redefine destination at job or sheet level.

**Q: Does this support CSV or other formats?**
This framework is Excel-focused but can be extended to CSV and Google Sheets.

**Q: How is schema evolution handled?**
Through BigQuery load options and framework-level checks.

---

**End of Document**
