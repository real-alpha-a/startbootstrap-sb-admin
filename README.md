import pandas as pd

def get_multi_row_concat_headers_pd(file_path, sheet_name, group_row_indices, header_row_idx, use_cols=None, separator="_"):
    """
    use_cols: can be a list of strings ["A", "B", "Z"] or indices [0, 1, 25]
    """
    # 1. Determine the range of rows to read
    start_row = min(group_row_indices)
    row_count = (header_row_idx - start_row) + 1
    
    # 2. Read only the header block and specific columns
    # This is the 'Fast' part: we ignore the rest of the sheet
    df_headers = pd.read_excel(
        file_path, 
        sheet_name=sheet_name, 
        header=None, 
        skiprows=start_row, 
        nrows=row_count,
        usecols=use_cols
    )

    # 3. Horizontal ffill (Handles Merged Columns)
    # axis=1 fills from left to right across the header rows
    df_headers = df_headers.ffill(axis=1)

    # 4. Vertical Redundancy Check
    # If a cell is vertically merged, row 0 and row 1 will be identical.
    # We clear the duplicate so we don't get "Sales_Sales"
    for i in range(len(df_headers) - 1):
        mask = df_headers.iloc[i].astype(str).strip() == df_headers.iloc[i+1].astype(str).strip()
        df_headers.iloc[i, mask] = ""

    # 5. Build Final Header Strings
    final_columns = []
    for col_i in range(df_headers.shape[1]):
        # Extract column, convert to string, remove 'nan' and 'Unnamed'
        parts = [str(x).strip() for x in df_headers.iloc[:, col_i] if pd.notna(x)]
        
        clean_parts = []
        for p in parts:
            if p.lower() != 'nan' and "unnamed:" not in p.lower() and p != "":
                # Append only if it's not a repeat of the last part added
                if not clean_parts or p != clean_parts[-1]:
                    clean_parts.append(p)
        
        final_columns.append(separator.join(clean_parts))

    return final_columns
