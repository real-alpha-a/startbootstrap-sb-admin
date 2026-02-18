import re
from typing import Dict


class MetadataExtractor:

    def __init__(self, config: Dict):
        self.file_delimiter = config.get("file_delimiter", "_")
        self.sheet_delimiter = config.get("sheet_delimiter", "_")
        self.fields = config.get("fields", {})

    def _split(self, value: str, delimiter: str):
        return value.split(delimiter) if delimiter else [value]

    def _resolve_segment(self, segments, key):
        if key.endswith("-1"):
            return segments[-1] if segments else ""

        index = int(key[4:])
        return segments[index] if index < len(segments) else ""

    def extract(self, filename: str, sheet_name: str) -> Dict:

        file_segments = self._split(filename, self.file_delimiter)
        sheet_segments = self._split(sheet_name, self.sheet_delimiter)

        variables = {
            "file": filename,
            "sheet": sheet_name,
            "file_sheet": f"{filename}_{sheet_name}",
        }

        # Add file segments
        for i in range(len(file_segments)):
            variables[f"fseg{i}"] = file_segments[i]
        variables["fseg-1"] = file_segments[-1] if file_segments else ""

        # Add sheet segments
        for i in range(len(sheet_segments)):
            variables[f"sseg{i}"] = sheet_segments[i]
        variables["sseg-1"] = sheet_segments[-1] if sheet_segments else ""

        metadata = {}

        for field, template in self.fields.items():

            # If no placeholder → direct value
            if "{" not in template:
                metadata[field] = template
                continue

            value = template
            for key, val in variables.items():
                value = value.replace(f"{{{key}}}", val)

            # Remove unreplaced placeholders
            value = re.sub(r"\{.*?\}", "", value)

            metadata[field] = value

        return metadata



# =========================================================
# METADATA CONFIG
# =========================================================
metadata:

  file_delimiter: "_"
  sheet_delimiter: " "

  # Available variables:
  #   {file}
  #   {sheet}
  #   {file_sheet}
  #
  #   {fseg0}, {fseg1}, {fseg-1}
  #   {sseg0}, {sseg1}, {sseg-1}
  #
  fields:

    # direct constant
    layer: "landing"

    # full filename with extension
    file_name: "{file}"

    # segments
    domain: "{fseg0}"
    year: "{fseg1}"

    sheet_type: "{sseg0}"

    # mixed
    identifier: "{fseg1}_{sseg1}"

    # combined
    full_identifier: "{file_sheet}"
