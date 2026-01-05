

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






import yaml
import os
import logging
from typing import Dict, Any, List, Optional, Union
from pathlib import Path

# Setup basic logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

config = None

class ConfigManager:
    _instance = None
    _config_data = {}
    _job_index = {}

    def __new__(cls, config_path: str = "./configuration/ingestion_config.yaml"):
        """
        Singleton Pattern: Ensures config is loaded only once per runtime.
        """
        if cls._instance is None:
            cls._instance = super(ConfigManager, cls).__new__(cls)
            cls._instance._initialized = False
        return cls._instance

    def __init__(self, config_path: str = "configurations/ingestion_config.yaml"):
        if self._initialized:
            return
            
        base_dir = os.path.dirname(os.path.abspath(__file__))        
        self.config_path = os.path.join(base_dir, config_path)
        self.reload_config()
        self._initialized = True

    def reload_config(self):
        """Loads or reloads the YAML configuration file."""
        if not os.path.exists(self.config_path):
            raise FileNotFoundError(f"Configuration file not found at: {self.config_path}")

        try:
            with open(self.config_path, 'r') as f:
                self._config_data = yaml.safe_load(f)
                
            # Create an index for fast O(1) job lookups
            self._job_index = {
                job['job_id']: job 
                for job in self._config_data.get('jobs', [])
            }
            logger.info(f"Configuration loaded successfully from {self.config_path}")
            
        except yaml.YAMLError as e:
            logger.error(f"Failed to parse YAML: {e}")
            raise

    # =========================================================================
    # 1. GLOBAL GETTERS
    # =========================================================================
    
    @property
    def project_id(self) -> str:
        return self._config_data['global_config']['bq_project_id']

    @property
    def metadata_cols(self) -> Dict[str, str]:
        return self._config_data['global_config'].get('metadata_columns', {})

    def get_global_bucket(self) -> str:
        return self._config_data['global_config'].get('bucket')

    # =========================================================================
    # 2. JOB CONTEXT (Bucket Resolution)
    # =========================================================================

    def get_job_config(self, job_id: str) -> Dict[str, Any]:
        """
        Retrieves job details and RESOLVES the bucket (Job > Global).
        """
        job = self._job_index.get(job_id)
        if not job:
            raise ValueError(f"Job ID '{job_id}' not found in configuration.")

        # Resolve Bucket: Check Job specific input first, then Global default
        job_specific_bucket = job.get('gcs_input', {}).get('bucket')
        global_bucket = self.get_global_bucket()
        
        final_bucket = job_specific_bucket if job_specific_bucket else global_bucket

        if not final_bucket:
            raise ValueError(f"Bucket not defined for job '{job_id}' and no global default found.")

        return {
            "job_id": job_id,
            "description": job.get("description"),
            "bucket": final_bucket,
            "file_selector": job.get("gcs_input", {}).get("file_selector"),
            "post_processing": job.get("gcs_input", {}).get("post_processing", {}),
            "target_defaults": job.get("target_defaults", {})
        }

    # =========================================================================
    # 3. HIERARCHICAL PARSING RULES (The "Smart Merge")
    # =========================================================================

    def get_sheet_config(self, job_id: str, sheet_identifier: str) -> Dict[str, Any]:
        """
        Returns the FULL configuration for a specific sheet.
        Merges: Global Rules -> Job Rules -> Sheet Rules
        """
        job = self._job_index.get(job_id)
        if not job:
            raise ValueError(f"Job ID '{job_id}' not found.")

        # Find the specific sheet entry
        sheet_entry = next((s for s in job.get('sheets', []) if s['sheet_identifier'] == sheet_identifier), None)
        if not sheet_entry:
            raise ValueError(f"Sheet '{sheet_identifier}' not found in job '{job_id}'.")

        # --- MERGE LOGIC START ---
        
        # 1. Start with Global Defaults
        final_rules = self._config_data['global_config'].get('default_parsing_rules', {}).copy()

        # 2. Overlay Job Rules
        job_rules = job.get('job_parsing_rules', {})
        final_rules = self._smart_update(final_rules, job_rules)

        # 3. Overlay Sheet Rules
        sheet_rules = sheet_entry.get('sheet_parsing_rules', {})
        final_rules = self._smart_update(final_rules, sheet_rules)
        
        # --- MERGE LOGIC END ---

        # Resolve Target Table and Dataset
        dataset = sheet_entry.get('dataset') or job.get('target_defaults', {}).get('dataset') or self._config_data['global_config']['bq_dataset_id']
        
        return {
            "sheet_identifier": sheet_identifier,
            "target_dataset": dataset,
            "target_table": sheet_entry.get('target_table'),
            "parsing_rules": final_rules,
            "schema_config": sheet_entry.get('schema_config', None) # explicit schema if present
        }

    def _smart_update(self, base: Dict, override: Dict) -> Dict:
        """
        Helper to merge dictionaries. 
        Note: For lists (like force_string_cols), this performs a UNION (Combine),
        not a replacement. This ensures specific jobs don't accidentally lose global string rules.
        """
        result = base.copy()
        for key, value in override.items():
            # Special handling for 'force_string_cols' -> Union the lists
            if key == "force_string_cols" and isinstance(value, list) and isinstance(result.get(key), list):
                # Combine distinct values from both lists
                result[key] = list(set(result[key]) | set(value))
            else:
                # Standard overwrite
                result[key] = value
        return result

# Global Instance for easy import
# Usage in other files: `from config_manager import config`
try:
    config = ConfigManager()
except Exception as e:
    logger.warning(f"Config auto-load failed (Ignore if running unit tests): {e}")
