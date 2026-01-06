import re
import pandas as pd
from typing import List

def clean_bq_header(header: str) -> str:
    """
    Sanitizes a string to be a valid BigQuery column name.
    Rules:
      - Alphanumeric and underscores only.
      - Max length 300.
      - Cannot start with a number.
    """
    if not header:
        return "col_unknown"

    # 1. Lowercase and replace spaces/special chars with underscores
    #    Regex matches anything NOT alphanumeric or underscore
    clean = re.sub(r'[^0-9a-zA-Z_]', '_', str(header).strip().lower())

    # 2. Collapse multiple underscores (e.g., "Order # Amount" -> "order___amount" -> "order_amount")
    clean = re.sub(r'_+', '_', clean)

    # 3. Strip trailing/leading underscores (optional, but cleaner)
    clean = clean.strip('_')

    # 4. Ensure it starts with a letter or underscore (BQ rule)
    #    If it starts with a digit (e.g., "2024_sales"), prepend an underscore ("_2024_sales")
    if clean and clean[0].isdigit():
        clean = f"_{clean}"

    # 5. Handle empty string (if header was just "###")
    if not clean:
        return "col_unknown"

    # 6. Truncate to 300 chars (BQ limit)
    return clean[:300]

def sanitize_dataframe_columns(df: pd.DataFrame) -> pd.DataFrame:
    """
    Renames all dataframe columns to match BQ standards.
    Handles duplicates (e.g., 'id', 'id' -> 'id', 'id_1').
    """
    new_cols = []
    seen_cols = {}

    for col in df.columns:
        clean_col = clean_bq_header(col)

        # Handle Duplicates
        if clean_col in seen_cols:
            seen_cols[clean_col] += 1
            clean_col = f"{clean_col}_{seen_cols[clean_col]}"
        else:
            seen_cols[clean_col] = 0
        
        new_cols.append(clean_col)

    df.columns = new_cols
    return df

# ==========================================
# Example Usage
# ==========================================
if __name__ == "__main__":
    # Simulate a messy Excel file
    data = {
        "Order ID": [1],
        "Total Value ($)": [100],
        "2024 Budget": [5000],
        "  Tax %  ": [5],
        "Comments / Notes": ["Test"],
        "Order ID": [2] # Duplicate
    }
    
    # Create DF (Pandas will auto-rename duplicate to 'Order ID.1', but let's assume raw)
    df = pd.DataFrame(data)
    # Force duplicate for demo (Pandas deduplicates on read, but this ensures safety)
    df.columns = ["Order ID", "Total Value ($)", "2024 Budget", "  Tax %  ", "Comments / Notes", "Order ID"]

    print("Before:", df.columns.tolist())
    
    df_clean = sanitize_dataframe_columns(df)
    
    print("After: ", df_clean.columns.tolist())
