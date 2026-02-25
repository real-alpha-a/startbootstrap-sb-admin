import pandas as pd

def get_merged_headers(
    file_path,
    group_header_row,
    header_row,
    skip_first_n_cols=0
):
    """
    Returns merged headers only.
    Row numbers are 1-based (Excel style).
    """

    # Convert to 0-based
    group_idx = group_header_row - 1
    header_idx = header_row - 1

    # We only need rows up to max(header_row)
    max_row = max(group_idx, header_idx) + 1

    df = pd.read_excel(
        file_path,
        header=None,
        nrows=max_row
    )

    group_header = df.iloc[group_idx].ffill()
    header = df.iloc[header_idx]

    merged_headers = []
    unnamed_count = 0

    for g, h in zip(group_header, header):
        g = str(g).strip() if pd.notna(g) else ""
        h = str(h).strip() if pd.notna(h) else ""

        if h == "" or h.lower().startswith("unnamed"):
            h = f"unnamed_{unnamed_count}"
            unnamed_count += 1

        if g:
            merged_headers.append(f"{g}_{h}")
        else:
            merged_headers.append(h)

    # Skip first N columns
    if skip_first_n_cols > 0:
        merged_headers = merged_headers[skip_first_n_cols:]

    return merged_headers
