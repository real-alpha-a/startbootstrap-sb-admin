import pandas as pd
import re
from typing import List


def make_bq_compatible(column_name: str) -> str:
    column_name = column_name.lower().strip()
    column_name = re.sub(r"[^\w]", "_", column_name)
    column_name = re.sub(r"_+", "_", column_name)
    column_name = column_name.strip("_")

    if not column_name:
        return ""

    if not re.match(r"^[a-zA-Z_]", column_name):
        column_name = f"col_{column_name}"

    return column_name[:300]


def merge_multi_group_headers(
    file_path: str,
    group_header_rows: List[int],
    header_row: int,
    sheet_name: str = 0
) -> pd.DataFrame:

    # Identify first relevant row
    first_required_row = min(group_header_rows)

    # Read only required portion
    df_raw = pd.read_excel(
        file_path,
        header=None,
        sheet_name=sheet_name,
        skiprows=first_required_row
    )

    # Adjust row indexes because we skipped rows
    adjusted_group_rows = [r - first_required_row for r in group_header_rows]
    adjusted_header_row = header_row - first_required_row

    total_cols = df_raw.shape[1]
    final_columns = []
    unnamed_counter = 0

    for col_idx in range(total_cols):
        parts = []

        # Merge group headers
        for row in adjusted_group_rows:
            value = df_raw.iloc[row, col_idx]
            if pd.notna(value) and str(value).strip():
                parts.append(str(value).strip())

        # Add actual header row
        header_value = df_raw.iloc[adjusted_header_row, col_idx]
        if pd.notna(header_value) and str(header_value).strip():
            parts.append(str(header_value).strip())

        combined = "_".join(parts)
        combined = make_bq_compatible(combined)

        if not combined:
            combined = f"unnamed_{unnamed_counter}"
            unnamed_counter += 1

        final_columns.append(combined)

    # Handle duplicates
    seen = {}
    unique_columns = []

    for col in final_columns:
        if col not in seen:
            seen[col] = 0
            unique_columns.append(col)
        else:
            seen[col] += 1
            unique_columns.append(f"{col}_{seen[col]}")

    # Data starts after header row
    df_final = df_raw.iloc[adjusted_header_row + 1:].copy()
    df_final.columns = unique_columns
    df_final.reset_index(drop=True, inplace=True)

    return df_final
