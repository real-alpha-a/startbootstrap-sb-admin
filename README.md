# =============================================================================
# EXCEL-TO-BIGQUERY INGESTION TEMPLATE
# =============================================================================
# This template defines how to parse Excel files and load them into BigQuery.
# 
# CONFIGURATION HIERARCHY (Priority):
# 1. Sheet-Specific Rules (Highest - overrides everything)
# 2. Job-Level Rules (Middle - overrides Global)
# 3. Global Parsing Defaults (Lowest - fallback)
# =============================================================================

global_config:
  gcp_project_id: "<GCP_PROJECT_ID>" # The target Google Cloud Project ID
  
  # Dataset destinations for the different layers of your Data Lake
  bq_land_dataset_id: "<LANDING_DATASET_NAME>"   # Raw, un-casted data
  bq_ingest_dataset_id: "<INGESTION_DATASET_NAME>" # Cleaned/Transformed data
  
  audit:
    dataset: "<LANDING_DATASET_NAME>"
    table: "<AUDIT_LOG_TABLE_NAME>" # Tracks file processing status, row counts, and timestamps

  loading_defaults:
    write_mode: "WRITE_TRUNCATE"     # WRITE_TRUNCATE (overwrites), WRITE_APPEND, or WRITE_EMPTY
    continue_on_error: false         # If true, one failed row won't stop the whole job
    skip_if_exists: false            # Set to true to avoid reprocessing the same file

  # Layer 1: Global Parsing Defaults
  default_parsing_rules:
    header_row: 0                    # 0-indexed row number where headers are found
    data_start_row: 1                # 0-indexed row where actual data begins
    trim_whitespace: true            # Removes leading/trailing spaces from strings
    treat_as_null: ["N/A", "NULL", "nan"] # Values to be converted to SQL NULL
    force_string_cols: []            # List column names to force as STRING (e.g., zip codes)

# =========================================================
# INGESTION JOBS (Define multiple jobs in this list)
# =========================================================
jobs:
  - job_id: "<UNIQUE_JOB_IDENTIFIER>"
    description: "<BRIEF_DESCRIPTION_OF_THE_PIPELINE>"
    
    # 1. GCS INPUT SETTINGS: Where the files live
    gcs_input:
      bucket: "<GCS_BUCKET_NAME>"
      
    file_selector:
      match_type: "regex"            # Use 'name' for direct match or 'regex' for patterns
      prefix: "incoming/"            # Root folder to scan
      supported_formats: "xlsx"
      pattern: "<REGEX_PATTERN_OR_FILENAME>" # e.g., "sales_.*\.xlsx"

    # 2. POST PROCESSING: What happens after a successful load?
    post_processing:
      action: "archive"              # Options: 'archive', 'delete', or 'none'
      archive_path: "processed_logs/" 

    # Layer 2: Job-Level Rules (Overrides Global Defaults for this job only)
    job_parsing_rules:
      header_row: 0
      data_start_row: 1

    # 3. SHEET CONFIGURATION: Define settings for specific tabs in the Excel file
    sheets:
      - sheet_identifier: 0          # Use the sheet index (0, 1, 2...) or the Sheet Name
        identifier_type: "index"     # 'index' or 'name'
        land_table: "<LANDING_TABLE_NAME>"
        ingest_table: "<FINAL_INGEST_TABLE_NAME>"

        # Layer 3: Sheet-Specific Rules (Highest Priority)
        parsing_rules:
          is_header_missing: false   # Set to true if Excel has no header row
          group_header:              # Used for multi-row/complex headers
            enabled: false
            rows: [0]                # Which rows contain header info to be merged
            separator: "_"           # Character to join multi-row headers
            direction: "prefix"      # Add prefix or postfix during merge
            ignore_labels: []        # Skip specific cells (e.g., "Report Date:")
          skip_footer: 0             # Number of rows to ignore at the bottom of the sheet
          # usecols: "A:M"           # Optional: Restrict parsing to specific Excel columns

        # Column Mapping: (Excel Header Name -> BigQuery Column Name)
        column_mapping:
          "<EXCEL_HEADER_NAME>": "<BQ_COLUMN_NAME>"

        # Schema Definition
        schema_config:
          mode: "auto_detect"        # 'auto_detect' (BQ decides) or 'explicit' (User defines)
          fields: []                 # If mode is 'explicit', define fields like:
                                     # - {name: "id", type: "INTEGER"}
