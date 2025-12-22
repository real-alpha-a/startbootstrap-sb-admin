version: "3.0"

# ==========================================
# TIER 1: SYSTEM GLOBAL DEFAULTS
# ==========================================
global_defaults:
  parsing:
    data_row_starts: 1
    data_row_ends: null
    has_header_in_range: true
    is_header_missing: false
    ordered_header: []
  destination:
    bq_dataset: "landing_zone"
    write_disposition: "WRITE_TRUNCATE"

# ==========================================
# TIER 2: FILE CONFIGURATIONS
# ==========================================
ingestion_jobs:

  # --------------------------------------------------------
  # JOB A: Monthly Sales (REGEX Match)
  # Matches files like: "sales_2024_01.xlsx", "sales_v2.xlsx"
  # --------------------------------------------------------
  - job_name: "sales_ingestion"
    
    # NEW: File Selector Block
    file_selector:
      pattern: "^sales_.*\.xlsx$"
      type: "regex"  # Options: 'regex' or 'exact'

    # File-Level Defaults (Tier 2)
    file_defaults:
      bq_dataset: "sales_mart" 

    # Sheet-Level Rules (Tier 3)
    sheet_rules:
      - selector: "Summary"
        selector_type: "exact"
        bq_table_name: "monthly_summary"
        parsing:
          data_row_starts: 5
          column_range: "A:F"

  # --------------------------------------------------------
  # JOB B: Master Mapping File (EXACT Match)
  # Matches ONLY: "master_product_list.xlsx"
  # --------------------------------------------------------
  - job_name: "master_data"
    
    file_selector:
      pattern: "master_product_list.xlsx"
      type: "exact"

    file_defaults:
      bq_dataset: "ref_data"
      parsing:
        has_header_in_range: true

    sheet_rules:
      # If the file has sheets like "US_Codes", "EU_Codes"
      - selector: ".*_Codes" 
        selector_type: "regex"
        bq_table_name: "geo_codes"



import yaml
import re
import os

class ConfigEngine:
    def __init__(self, config_path):
        with open(config_path, 'r') as f:
            self.full_config = yaml.safe_load(f)
        
        self.system_globals = self.full_config.get('global_defaults', {})
        self.jobs = self.full_config.get('ingestion_jobs', [])

    def _merge_configs(self, base, override):
        """
        Deep merges two configuration dictionaries (Base + Override).
        Logic: If key exists in override, use it. Otherwise keep base.
        Handles nested dictionaries (like 'parsing' or 'destination').
        """
        merged = base.copy()
        for key, value in override.items():
            if isinstance(value, dict) and key in merged:
                merged[key] = self._merge_configs(merged[key], value)
            else:
                merged[key] = value
        return merged

    def _is_match(self, target_string, selector_config):
        """
        Generic matcher for both Files and Sheets.
        """
        pattern = selector_config.get('pattern') or selector_config.get('selector') # Handle naming diffs
        match_type = selector_config.get('type') or selector_config.get('selector_type', 'exact')

        if match_type == 'regex':
            return bool(re.search(pattern, target_string))
        elif match_type == 'exact':
            return target_string == pattern
        return False

    def get_job_for_file(self, filename):
        """
        Iterates through all jobs to find which one matches this filename.
        Returns the Job Config if found, else None.
        """
        for job in self.jobs:
            selector = job.get('file_selector', {})
            if self._is_match(filename, selector):
                return job
        return None

    def get_final_config(self, filename, sheet_name):
        """
        THE MAGIC FUNCTION:
        Returns the final merged configuration for a specific Sheet in a specific File.
        """
        # 1. Find the Job (File match)
        job = self.get_job_for_file(filename)
        if not job:
            return None # File is not configured to be ingested

        # 2. Start with System Globals (Tier 1)
        current_config = self.system_globals.copy()

        # 3. Merge File Defaults (Tier 2)
        file_defaults = job.get('file_defaults', {})
        current_config = self._merge_configs(current_config, file_defaults)

        # 4. Find Sheet Rule (Tier 3)
        sheet_rules = job.get('sheet_rules', [])
        sheet_specific_config = {}
        
        matched_rule = None
        for rule in sheet_rules:
            # We treat the rule itself as the selector config
            if self._is_match(sheet_name, rule):
                matched_rule = rule
                # We stop at the FIRST match (priority)
                break
        
        # 5. Merge Sheet Overrides if found
        if matched_rule:
            # Add table name if it exists in the rule
            if 'bq_table_name' in matched_rule:
                if 'destination' not in current_config: current_config['destination'] = {}
                current_config['destination']['bq_table_name'] = matched_rule['bq_table_name']

            # Merge parsing/destination overrides
            sheet_specific_config = matched_rule.get('parsing', {})
            if sheet_specific_config:
                current_config['parsing'] = self._merge_configs(current_config.get('parsing', {}), sheet_specific_config)

        # Return the final flattened config ready for the converter
        return current_config

# --- USAGE EXAMPLE ---
# engine = ConfigEngine("ingestion_config.yaml")

# # Scenario: We are processing 'sales_2024.xlsx', sheet 'Summary'
# config = engine.get_final_config("sales_2024.xlsx", "Summary")

# print(f"Row Start: {config['parsing']['data_row_starts']}") 
# # Output: 5 (Inherited from sheet rule)

# print(f"Dataset: {config['destination']['bq_dataset']}")
# # Output: sales_mart (Inherited from file default)
