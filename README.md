import re
from typing import Dict, Optional


class TableNameGenerator:
    """
    Generates table names based on configuration.

    Resolution order:
        1. Remove file extension
        2. Apply file alias (case-insensitive)
        3. Split filename using file_delimiter
        4. Apply sheet alias (case-insensitive)
        5. Split sheet using sheet_delimiter
        6. Apply final_template
    """

    def __init__(self, config: Dict):
        self.strategy = config.get("strategy", "template")
        self.file_delimiter = config.get("file_delimiter", "_")
        self.sheet_delimiter = config.get("sheet_delimiter", "_")
        self.final_template = config.get("final_template", "{file}_{sheet}")

        self.file_alias = self._normalize_keys(config.get("file_alias", {}))
        self.sheet_alias = self._normalize_keys(config.get("sheet_alias", {}))

    @staticmethod
    def _normalize_keys(mapping: Dict[str, str]) -> Dict[str, str]:
        """Lowercase keys for case-insensitive matching."""
        return {k.lower(): v for k, v in mapping.items()}

    @staticmethod
    def _remove_extension(filename: str) -> str:
        return filename.rsplit(".", 1)[0]

    def _apply_alias(self, name: str, alias_map: Dict[str, str]) -> str:
        return alias_map.get(name.lower(), name)

    def _split_segments(self, value: str, delimiter: str):
        if delimiter == "":
            return [value]
        return value.split(delimiter)

    def _resolve_segment(self, segments, index: int) -> str:
        try:
            return segments[index]
        except IndexError:
            return ""

    def generate(self, filename: str, sheet_name: str) -> str:
        if self.strategy == "file_only":
            return self._remove_extension(filename)

        # --- File processing ---
        base_file = self._remove_extension(filename)
        logical_file = self._apply_alias(base_file, self.file_alias)
        file_segments = self._split_segments(logical_file, self.file_delimiter)

        # --- Sheet processing ---
        logical_sheet = self._apply_alias(sheet_name, self.sheet_alias)
        sheet_segments = self._split_segments(logical_sheet, self.sheet_delimiter)

        if self.strategy == "file_sheet":
            return f"{logical_file}_{logical_sheet}"

        # --- Template processing ---
        variables = {
            "file": logical_file,
            "sheet": logical_sheet,
        }

        # File segments
        for i in range(len(file_segments)):
            variables[f"fseg{i}"] = file_segments[i]
        variables["fseg-1"] = self._resolve_segment(file_segments, -1)

        # Sheet segments
        for i in range(len(sheet_segments)):
            variables[f"sseg{i}"] = sheet_segments[i]
        variables["sseg-1"] = self._resolve_segment(sheet_segments, -1)

        # Replace variables safely
        result = self.final_template
        for key, value in variables.items():
            result = result.replace(f"{{{key}}}", value)

        # Clean unreplaced placeholders
        result = re.sub(r"\{.*?\}", "", result)

        # Optional: normalize multiple underscores
        result = re.sub(r"_+", "_", result).strip("_")

        return result







# =========================================================
# TABLE NAMING CONFIGURATION
# =========================================================
table_naming:

  # -------------------------------------------------------
  # Strategy
  # -------------------------------------------------------
  # file_only   → use filename only
  # file_sheet  → filename + sheet
  # template    → use final_template
  strategy: "template"


  # -------------------------------------------------------
  # File Segmentation
  # -------------------------------------------------------
  file_delimiter: "_"


  # -------------------------------------------------------
  # Sheet Segmentation
  # -------------------------------------------------------
  sheet_delimiter: " "


  # -------------------------------------------------------
  # Final Template
  # -------------------------------------------------------
  # FILE VARIABLES:
  #   {file}
  #   {fseg0}, {fseg1}, {fseg-1}
  #
  # SHEET VARIABLES:
  #   {sheet}
  #   {sseg0}, {sseg1}, {sseg-1}
  #
  # Hardcoded values allowed anywhere.
  #
  final_template: "{fseg0}_{sseg0}_landing"


  # -------------------------------------------------------
  # File Alias Override
  # -------------------------------------------------------
  file_alias:
    "Financial Sample": "finance_sample"
    "Sales Master 2025": "sales_master_2025"


  # -------------------------------------------------------
  # Sheet Alias Override
  # -------------------------------------------------------
  sheet_alias:
    "Summary Report FY25": "summary_fy25"
    "Raw Data Sheet": "raw_data"




import pytest
from tablename_generator import TableNameGenerator


@pytest.fixture
def base_config():
    return {
        "strategy": "template",
        "file_delimiter": "_",
        "sheet_delimiter": " ",
        "final_template": "{fseg0}_{sseg0}_landing",
        "file_alias": {
            "Financial Sample": "finance_sample"
        },
        "sheet_alias": {
            "Summary Report FY25": "summary_fy25"
        }
    }


def test_basic_template(base_config):
    gen = TableNameGenerator(base_config)

    result = gen.generate(
        "sales_2025_q1_india.xlsx",
        "Raw Data Sheet"
    )

    assert result == "sales_Raw_landing"


def test_file_alias(base_config):
    gen = TableNameGenerator(base_config)

    result = gen.generate(
        "Financial Sample.xlsx",
        "Raw Data Sheet"
    )

    assert result == "finance_Raw_landing"


def test_sheet_alias(base_config):
    gen = TableNameGenerator(base_config)

    result = gen.generate(
        "sales_2025.xlsx",
        "Summary Report FY25"
    )

    assert result == "sales_summary_fy25_landing"


def test_negative_segments():
    config = {
        "strategy": "template",
        "file_delimiter": "_",
        "sheet_delimiter": "_",
        "final_template": "{fseg-1}_{sseg-1}"
    }

    gen = TableNameGenerator(config)

    result = gen.generate(
        "sales_2025_q1_india.xlsx",
        "region_west"
    )

    assert result == "india_west"


def test_file_only_strategy():
    config = {"strategy": "file_only"}
    gen = TableNameGenerator(config)

    result = gen.generate("sales_2025.xlsx", "Sheet1")

    assert result == "sales_2025"


def test_file_sheet_strategy():
    config = {"strategy": "file_sheet"}
    gen = TableNameGenerator(config)

    result = gen.generate("sales.xlsx", "Sheet1")

    assert result == "sales_Sheet1"
