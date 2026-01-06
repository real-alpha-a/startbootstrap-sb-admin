import pandas as pd
from pathlib import Path
from typing import Union, Dict, List, Optional

def convert_excel_to_csv(
    excel_path: str, 
    output_dir: str, 
    sheet_name: str, # We force a specific sheet here for clarity with the detailed options
    # --- Parsing Options ---
    is_header_missing: bool = False,
    header_row: int = 0,
    data_start_row: int = 1,     # 0-indexed: Row 1 is the 2nd row in Excel
    data_end_row: Optional[int] = None,
    ordered_header: Optional[List[str]] = None,
    usecols: Optional[Union[str, List[int]]] = None,
    skip_footer: int = 0
) -> str:
    """
    Converts a specific Excel sheet to CSV with granular parsing controls.
    """
    source_file = Path(excel_path)
    if not source_file.exists():
        raise FileNotFoundError(f"Source file not found: {excel_path}")

    out_path = Path(output_dir)
    out_path.mkdir(parents=True, exist_ok=True)

    # 1. Setup 'header' parameter
    # If header is missing, tell Pandas to not look for one (header=None).
    # Otherwise, specify which row contains the header.
    pd_header = None if is_header_missing else header_row

    # 2. Setup 'skiprows' logic
    # This is tricky: We want to skip rows BEFORE data starts, 
    # but we must preserve the header row if it exists.
    skip_rows_arg = None
    
    if is_header_missing:
        # Simple: Skip everything up to data_start
        if data_start_row > 0:
            skip_rows_arg = data_start_row
    else:
        # Complex: We need the header row, but might need to skip garbage between header and data.
        # Example: Header at 0, Data starts at 5. We keep 0, skip 1,2,3,4.
        gap_start = header_row + 1
        gap_end = data_start_row
        if gap_end > gap_start:
            skip_rows_arg = range(gap_start, gap_end)

    # 3. Setup 'nrows' (Data End)
    # Pandas doesn't have 'end_row', it has 'nrows' (count).
    pd_nrows = None
    if data_end_row is not None:
        pd_nrows = data_end_row - data_start_row

    # 4. Read Excel
    try:
        df = pd.read_excel(
            source_file,
            sheet_name=sheet_name,
            header=pd_header,
            skiprows=skip_rows_arg,
            nrows=pd_nrows,
            usecols=usecols,
            skipfooter=skip_footer,
            dtype=str # Force all to string for Bronze Layer safety
        )
    except Exception as e:
        raise ValueError(f"Error reading Excel sheet '{sheet_name}': {e}")

    # 5. Apply 'ordered_header' (Renaming columns)
    if ordered_header:
        # Validation: Column count must match
        if len(ordered_header) != len(df.columns):
            raise ValueError(
                f"Config mismatch: 'ordered_header' has {len(ordered_header)} names, "
                f"but file has {len(df.columns)} columns."
            )
        df.columns = ordered_header

    # 6. Save to CSV
    clean_sheet = sheet_name.replace(" ", "_")
    csv_filename = f"{source_file.stem}_{clean_sheet}.csv"
    csv_full_path = out_path / csv_filename

    df.to_csv(
        csv_full_path, 
        index=False, 
        encoding='utf-8', 
        quoting=1 # QUOTE_ALL
    )
    
    return str(csv_full_path)

# ==========================================
# Example Usage (Mapping from your YAML)
# ==========================================
if __name__ == "__main__":
    # Hypothetical Config from YAML
    config = {
        "is_header_missing": True,
        "data_start": 5,           # Data actually starts at row 6 (index 5)
        "data_end": 500,           # Stop at row 500
        "usecols": "A:D",          # Only read first 4 columns
        "ordered_header": ["sku", "qty", "loc", "mgr"], # Rename them
        "skip_footer": 2           # Ignore last 2 lines
    }

    try:
        csv_path = convert_excel_to_csv(
            excel_path="data.xlsx",
            output_dir="staging",
            sheet_name="APAC_Legacy",
            
            # Map Config to Arguments
            is_header_missing=config['is_header_missing'],
            data_start_row=config['data_start'],
            data_end_row=config['data_end'],
            usecols=config['usecols'],
            ordered_header=config['ordered_header'],
            skip_footer=config['skip_footer']
        )
        print(f"Success! CSV created at: {csv_path}")
        
    except Exception as e:
        print(f"Conversion Failed: {e}")
