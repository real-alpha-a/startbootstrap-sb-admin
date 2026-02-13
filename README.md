# -----------------------------------------
# Naming Strategy Registry
# -----------------------------------------
naming_strategies:

  simple_segment:

    # ----- FILE PARSING -----
    file:
      delimiter: "_"              # Split filename by "_"
      strip_extension: true       # Remove .xlsx

      # Example:
      # FLJAC_VTE_2024_Summary_2025-12-09.xlsx
      # After split:
      # [0]=FLJAC, [1]=VTE, [2]=2024, [3]=Summary, [4]=2025-12-09
      segments:
        ministry: [0]             # -> FLJAC
        report: [1]               # -> VTE

    # ----- SHEET PARSING -----
    sheet:
      delimiter: " "

      # If sheet matches, override segment
      # Example:
      # "VTE Performance Summary" -> perf_summ
      override:
        - match: "VTE Performance Summary"
          value: "perf_summ"

      # If no override, use segment logic
      # Example:
      # VTE Performance Summary
      # [0]=VTE, [1]=Performance, [2]=Summary
      segments:
        detail: [1, 2]            # -> Performance Summary

    # ----- FINAL OUTPUT -----
    output:
      order: [ministry, report, detail]
      join_with: "_"
      layer_suffix:
        land: "_land"

    # ----- NORMALIZATION -----
    normalize:
      lowercase: true
      replace_space: "_"



import os
import re


class TableNameGenerator:

    def __init__(self, strategy_config, layer="land"):
        self.config = strategy_config
        self.layer = layer

    def generate(self, file_name: str, sheet_name: str) -> str:
        file_segments = self._parse_file(file_name)
        sheet_value = self._parse_sheet(sheet_name)

        parts = []
        for key in self.config["output"]["order"]:
            if key in file_segments:
                parts.append(file_segments[key])
            elif key == "detail":
                parts.append(sheet_value)

        table_name = self.config["output"]["join_with"].join(parts)

        # Add layer suffix
        suffix = self.config["output"]["layer_suffix"].get(self.layer, "")
        table_name = table_name + suffix

        return self._normalize(table_name)

    # -----------------------------
    # FILE PARSER
    # -----------------------------
    def _parse_file(self, file_name: str):
        if self.config["file"].get("strip_extension", False):
            file_name = os.path.splitext(file_name)[0]

        tokens = file_name.split(self.config["file"]["delimiter"])
        segments = {}

        for name, positions in self.config["file"]["segments"].items():
            value = "_".join(tokens[i] for i in positions if i < len(tokens))
            segments[name] = value

        return segments

    # -----------------------------
    # SHEET PARSER
    # -----------------------------
    def _parse_sheet(self, sheet_name: str):

        # Check override
        for rule in self.config["sheet"].get("override", []):
            if rule["match"].lower() == sheet_name.lower():
                return rule["value"]

        tokens = sheet_name.split(self.config["sheet"]["delimiter"])

        positions = self.config["sheet"]["segments"]["detail"]
        return "_".join(tokens[i] for i in positions if i < len(tokens))

    # -----------------------------
    # NORMALIZATION
    # -----------------------------
    def _normalize(self, value: str) -> str:

        if self.config["normalize"].get("replace_space"):
            value = value.replace(" ", self.config["normalize"]["replace_space"])

        if self.config["normalize"].get("lowercase"):
            value = value.lower()

        value = re.sub(r"[^a-zA-Z0-9_]", "", value)
        value = re.sub(r"_+", "_", value)

        return value.strip("_")



strategy = config["naming_strategies"]["simple_segment"]

generator = TableNameGenerator(strategy, layer="land")

table = generator.generate(
    "FLJAC_VTE_2024_Summary_2025-12-09.xlsx",
    "VTE Performance Summary"
)

print(table)
