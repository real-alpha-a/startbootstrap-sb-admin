

# =============================================================================
# 1. GLOBAL DEFAULTS
# =============================================================================
global_config:
  bq_project_id: "your-gcp-project-id"
  bq_dataset_id: "raw_landing_zone"
  
  # Metadata columns to append to every table for lineage
  metadata_columns:
    ingestion_timestamp: "_ingestion_ts"
    source_filename: "_source_file"
    sheet_name: "_sheet_name"

  # Default Loading Strategy
  write_mode: "WRITE_TRUNCATE" 

  # Layer 1: Global Defaults (Lowest Priority)
  default_parsing_rules:
    header_row: 0          # 0-indexed (Row 1 in Excel)
    data_start: 1          # 0-indexed (Row 2 in Excel)
    trim_whitespace: true
    treat_as_null: ["N/A", "NULL", "-", "", "nan"]
    
    # Critical for Excel: Force specific columns to read as String to prevent
    # losing leading zeros (e.g., zip codes "00123" -> 123)
    # This is a global fallback; specific jobs usually override this.
    force_string_cols: [] 

# =============================================================================
# 2. INGESTION JOBS
# =============================================================================
jobs:
  - job_id: "inventory_master_upload"
    description: "Inventory file processing"

    gcs_input:
      bucket: "landing-zone-bucket" # Added explicit bucket
      file_selector:
        match_type: "regex"
        pattern: "^inventory/stock_levels_v.*\\.xlsx$"
      
      # Action to take on source file after success
      post_processing:
        action: "archive"  # Options: archive, delete, move
        archive_path: "inventory/archive/"

    # Layer 2: Job-Level Rules (Middle Priority)
    job_parsing_rules:
      header_row: 0
      data_start: 1
      is_header_missing: false
      # Ensure SKUs are always treated as strings to keep formatting
      force_string_cols: ["sku_id", "material_code"] 

    # List of Sheets to process
    sheets:
      # Scenario A: Standard Sheet (Inherits Job Rules)
      - sheet_identifier: "US_Stock" # Renamed from 'sheet_name' to allow flexibility
        identifier_type: "name"      # Options: 'name', 'index', 'regex'
        target_table: "inv_us_stock"
        
        # Schema Enforcement: Explicitly define schema to prevent drift
        schema_config:
          mode: "explicit" # Options: 'auto_detect', 'explicit'
          fields:
            - {name: "sku_id", type: "STRING", mode: "REQUIRED"}
            - {name: "qty", type: "INTEGER"}
            - {name: "last_updated", type: "DATE"}

      # Scenario B: Complex Sheet (Overrides Job Rules)
      - sheet_identifier: "APAC_Legacy_Data"
        identifier_type: "name"
        target_table: "inv_apac_stock"
        
        # Layer 3: Sheet-Level Rules (Highest Priority)
        sheet_parsing_rules:
          is_header_missing: true
          header_row: null           # Explicitly nullify since we provide manual headers
          data_start: 0
          data_end: 500              
          # Renaming columns from Excel headers (A, B, C...) to DB friendly names
          ordered_header: ["sku_id", "qty", "location_code", "manager"]
          usecols: "A:D"             # Pandas standard is 'usecols', clearer than 'col_range'
          skip_footer: 2










import json
from config_manager import config

def print_separator(char="=", length=80):
    print(char * length)

def inspect_all_configs():
    """
    Loops through all jobs defined in the YAML and prints the 
    fully resolved configuration for each sheet.
    """
    print("STARTING CONFIGURATION INSPECTION...")
    print(f"Global Project ID: {config.project_id}")
    print(f"Global Metadata Columns: {config.metadata_cols}")
    print_separator()

    # Access the raw list of jobs from the loaded config data
    # We use the internal _config_data here to discover what jobs exist
    all_jobs = config._config_data.get('jobs', [])

    if not all_jobs:
        print("No jobs found in configuration!")
        return

    for job_entry in all_jobs:
        job_id = job_entry.get('job_id')
        
        # 1. Resolve Job Context (Bucket, Pattern, etc.)
        try:
            job_ctx = config.get_job_config(job_id)
        except ValueError as e:
            print(f"!! ERROR loading Job {job_id}: {e}")
            continue

        print(f"\nJOB ID: {job_id}")
        print(f"Description: {job_ctx['description']}")
        print(f"Source Bucket: '{job_ctx['bucket']}' (Resolved)")
        print(f"File Pattern:  '{job_ctx['file_selector']['pattern']}'")
        print(f"Post-Process:  {job_ctx['post_processing'].get('action', 'None')}")
        print("-" * 40)

        # 2. Loop through all sheets in this job
        sheets = job_entry.get('sheets', [])
        
        for sheet in sheets:
            sheet_identifier = sheet.get('sheet_identifier')
            
            # 3. Resolve Sheet Context (Parsing Rules, Schema, Target)
            try:
                sheet_ctx = config.get_sheet_config(job_id, sheet_identifier)
                
                print(f"  SHEET: {sheet_identifier}")
                print(f"  -> Target: {sheet_ctx['target_dataset']}.{sheet_ctx['target_table']}")
                
                # specific schema check
                if sheet_ctx['schema_config']:
                    print(f"  -> Schema Mode: {sheet_ctx['schema_config']['mode']} (Explicit schema defined)")
                else:
                    print(f"  -> Schema Mode: Auto-Detect")

                print("  -> Effective Parsing Rules (Merged):")
                
                # Pretty print the dictionary of rules
                rules_json = json.dumps(sheet_ctx['parsing_rules'], indent=6)
                print(rules_json)
                print("") # Empty line for spacing

            except ValueError as e:
                print(f"  !! ERROR loading Sheet {sheet_identifier}: {e}")

        print_separator("-")

if __name__ == "__main__":
    try:
        inspect_all_configs()
    except Exception as e:
        print(f"Critical Error: {e}")
