# pipeline_runner.py

import pandas as pd
import os
import re
from config_loader import load_configurations # Assuming config_loader.py is in the same directory
from typing import Dict, Any
  
# Define a global variable that will hold loaded configuration instance
global config

# Load the configuration manager
config = load_configurations(
    pipeline_path="/Users/atul/python-scripts/xlsx_to_bq/configurations/pipeline_config.yaml", # Assume this file is local for this example
)

def get_file_list(source_type: str, incoming_area: str, supported_formats: list) -> list:
    """
    Simulates fetching a list of files from either a local directory or GCS.
    For this example, we mock a list of files that match your regex.
    """
    
def run_read_and_filter(filename: str, file_config: Dict[str, Any]) -> pd.DataFrame | None:
    """
    STEP 1: Reads the source file (XLSX or CSV), applies column mapping,
    and returns a clean DataFrame.
    """
    
def run_load_to_landing(df: pd.DataFrame, landing_step_config: Dict[str, Any]):
    """
    STEP 2: Loads the processed DataFrame into the BigQuery landing table.
    """
    

# --- Main Execution Block ---

def main():
    """
    Main function to run the data ingestion pipeline.
    """
        
    # 1. Get the configuration steps
    read_step_config = next(s for s in config.pipeline_steps if s['step_name'] == 'read_and_filter')
    landing_step_config = next(s for s in config.pipeline_steps if s['step_name'] == 'load_to_landing')
    
    if not (read_step_config['enabled'] and landing_step_config['enabled']):
        print("INFO: Pipeline steps are disabled in config. Exiting.")
        return

    # 2. Get list of files to process (Simulation)
    source_type = config.global_settings['source_type']
    incoming_area = config.global_settings['incoming_area']
    supported_formats = read_step_config['parameters']['supported_formats']
    
    files_to_process = get_file_list(source_type, incoming_area, supported_formats)
    
    # 3. Process each file end-to-end
    all_processed_data = []
    
    for filename in files_to_process:
        # Get the specific configuration for this file
        file_config = config.get_config_for_file(filename)
        
        if file_config:
            # --- STEP 1: READ, FILTER, AND RENAME ---
            transformed_df = run_read_and_filter(filename, file_config)
            
            if transformed_df is not None and not transformed_df.empty:
                all_processed_data.append(transformed_df)
        else:
            print(f"INFO: Skipping file {filename} as no configuration match was found.")

    if not all_processed_data:
        print("\nPipeline finished: No data was successfully processed.")
        return

    # 4. Concatenate all processed dataframes for bulk loading
    final_df = pd.concat(all_processed_data, ignore_index=True)
    print(f"\n--- Consolidated {len(final_df)} rows for BigQuery Load ---")
    
    # --- STEP 2: LOAD TO LANDING DATASET ---
    run_load_to_landing(final_df, landing_step_config)
    
    print("\n--- Pipeline Run Complete ---")


if __name__ == "__main__":
    main()
