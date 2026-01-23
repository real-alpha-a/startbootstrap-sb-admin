def get_multi_row_concat_headers(file_path, sheet_name, group_row_indices, header_row_idx, separator="_"):
    wb = openpyxl.load_workbook(file_path, data_only=True)
    ws = wb[sheet_name]
    
    # 1. Get base headers from the primary header row
    df_header = pd.read_excel(file_path, sheet_name=sheet_name, header=None, 
                              skiprows=header_row_idx, nrows=1)
    sub_headers = [str(h).strip() for h in df_header.iloc[0].tolist()]
    
    # 2. Build a comprehensive map of all merged cells
    merge_map = {}
    for merged_range in ws.merged_cells.ranges:
        val = ws.cell(row=merged_range.min_row, column=merged_range.min_col).value
        for r in range(merged_range.min_row, merged_range.max_row + 1):
            for c in range(merged_range.min_col, merged_range.max_col + 1):
                merge_map[(r, c)] = {
                    'val': str(val).strip() if val else None,
                    'range': merged_range
                }

    final_columns = []
    h_excel_row = header_row_idx + 1

    # 3. Iterate through each column to build the hierarchy
    for col_i, sub_val in enumerate(sub_headers):
        col_idx = col_i + 1
        path_parts = []
        
        for g_idx in group_row_indices:
            g_excel_row = g_idx + 1
            
            # Check if cell is merged or standard
            cell_data = merge_map.get((g_excel_row, col_idx))
            if cell_data:
                # If it's a vertical merge that covers the sub-header row, we skip it
                # to avoid redundant names like "Date_Date"
                if cell_data['range'].max_row >= h_excel_row:
                    continue
                val = cell_data['val']
            else:
                val = ws.cell(row=g_excel_row, column=col_idx).value
            
            # Append only if value is present and not redundant
            clean_val = str(val).strip() if val and str(val).lower() != 'none' else None
            if clean_val and clean_val not in path_parts:
                path_parts.append(clean_val)
        
        # 4. Final Concatenation
        # Filter out 'Unnamed' artifacts from sub_val
        clean_sub = sub_val if "Unnamed:" not in sub_val else ""
        
        if clean_sub and clean_sub not in path_parts:
            path_parts.append(clean_sub)
            
        final_columns.append(separator.join(path_parts))

    return final_columns
