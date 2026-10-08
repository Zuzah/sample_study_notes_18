# sample_study_notes_18
KYC/AML Studies

**new** app/services/manifest_service.py:

```python
import csv
import json
from datetime import datetime, timezone
from pathlib import Path

from app.core.logging import log


class ManifestService:
    """Structured JSON manifest (.mrk) sidecar, for reports that need a
    downstream ingestion-framework manifest instead of a SHA256 checksum (see
    marker_service.py). Opt in via ReportDefinition.marker_type. One shape
    built so far (Fircosoft's contract) - a different shape gets its own
    create_*_for_file method here, not a parameter on this one."""

    @staticmethod
    def _record_count(csv_path: Path) -> int:
        with csv_path.open("r", encoding="utf-8", newline="") as stream:
            reader = csv.reader(stream)
            next(reader, None)  # header row
            return sum(1 for _ in reader)

    @classmethod
    def create_fircosoft_manifest_for_file(
        cls,
        source_file_path: Path,
        source_appl: dict,
        source_file_static: dict,
    ) -> Path:
        # dataFileURI = the real resolved filename, never an independently
        # templated string that could drift from what's actually delivered.
        source_file_path = Path(source_file_path)
        if not source_file_path.exists():
            raise FileNotFoundError(f"Source file not found: {source_file_path}")

        date_stamp = datetime.now(timezone.utc).strftime("%Y%m%d")
        record_count = cls._record_count(source_file_path)

        manifest = {
            "specVersion": "2.0",
            "sourceAppl": {**source_appl, "buste": date_stamp},
            "sourceFiles": [
                {
                    **source_file_static,
                    "recordCount": str(record_count),
                    "dataFileURI": source_file_path.name,
                }
            ],
        }

        manifest_path = Path(str(source_file_path) + ".mrk")
        manifest_path.write_text(
            json.dumps(manifest, indent=2) + "\n", encoding="utf-8"
        )
        log.info(f"Fircosoft manifest generated successfully: {manifest_path}")
        return manifest_path

```

tests/services/test_manifest_service.py:

```python
import json
from datetime import datetime, timezone
from pathlib import Path

import pytest

from app.services.manifest_service import ManifestService

SOURCE_APPL = {
    "providingParty": "gbm",
    "country": "can",
    "region": "nam",
    "appAcronym": "b8fb",
    "frequency": "dly",
    "securityClassification": "cpi",
    "operation": "f",
    "ingestionFramework": "y",
    "fileTransfer": "push",
}
SOURCE_FILE_STATIC = {
    "dataRention": "",
    "compressType": "",
    "ingestMetadata": "Fircosoft_EDL_Metadata_V8.xml",
    "fileCompressed": "n",
    "toCharSet": "",
    "recordCount": "",
    "dataFileMD5": "",
    "fromCharSet": "",
    "invalidRecordThreshold": "0",
    "fileExtension": "csv",
    "dataFileURI": "",
    "charSetConv": "n",
}


def _csv(tmp_path, rows):
    path = tmp_path / "Fircosoft_Fcore_EDL_Data_20261008.csv"
    lines = ["Fenergo ID,Legal Entity Name"] + rows
    path.write_text("\n".join(lines) + "\n")
    return path


def test_create_fircosoft_manifest_for_file_builds_correct_json(tmp_path):
    csv_path = _csv(tmp_path, ["1,Acme Corp", "2,Beta Inc", "3,Gamma LLC"])

    marker_path = ManifestService.create_fircosoft_manifest_for_file(
        csv_path, SOURCE_APPL, SOURCE_FILE_STATIC
    )

    manifest = json.loads(marker_path.read_text())
    today = datetime.now(timezone.utc).strftime("%Y%m%d")

    assert manifest["specVersion"] == "2.0"
    assert manifest["sourceAppl"]["buste"] == today
    assert manifest["sourceAppl"]["providingParty"] == "gbm"
    assert manifest["sourceAppl"]["appAcronym"] == "b8fb"
    source_file = manifest["sourceFiles"][0]
    assert source_file["recordCount"] == "3"
    assert source_file["dataFileURI"] == "Fircosoft_Fcore_EDL_Data_20261008.csv"
    assert source_file["ingestMetadata"] == "Fircosoft_EDL_Metadata_V8.xml"
    assert source_file["fileExtension"] == "csv"


def test_create_fircosoft_manifest_for_file_counts_only_data_rows_not_header(
    tmp_path,
):
    csv_path = _csv(tmp_path, ["1,Acme Corp"])

    marker_path = ManifestService.create_fircosoft_manifest_for_file(
        csv_path, SOURCE_APPL, SOURCE_FILE_STATIC
    )

    manifest = json.loads(marker_path.read_text())
    assert manifest["sourceFiles"][0]["recordCount"] == "1"


def test_create_fircosoft_manifest_for_file_dataFileURI_matches_real_filename(
    tmp_path,
):
    """dataFileURI must always equal the real delivered filename - never an
    independently-templated string that could drift from what's actually sent."""
    csv_path = tmp_path / "some_other_name.csv"
    csv_path.write_text("Fenergo ID\n1\n")

    marker_path = ManifestService.create_fircosoft_manifest_for_file(
        csv_path, SOURCE_APPL, SOURCE_FILE_STATIC
    )

    manifest = json.loads(marker_path.read_text())
    assert manifest["sourceFiles"][0]["dataFileURI"] == "some_other_name.csv"


def test_create_fircosoft_manifest_for_file_preserves_field_order(tmp_path):
    """recordCount/dataFileURI land at whatever position SOURCE_FILE_STATIC
    declares them at (toCharSet->recordCount, fileExtension->dataFileURI->
    charSetConv), matching the real sample manifest byte for byte - not
    appended at the end."""
    csv_path = _csv(tmp_path, ["1,Acme Corp"])

    marker_path = ManifestService.create_fircosoft_manifest_for_file(
        csv_path, SOURCE_APPL, SOURCE_FILE_STATIC
    )

    manifest = json.loads(marker_path.read_text())
    keys = list(manifest["sourceFiles"][0].keys())
    assert keys == [
        "dataRention",
        "compressType",
        "ingestMetadata",
        "fileCompressed",
        "toCharSet",
        "recordCount",
        "dataFileMD5",
        "fromCharSet",
        "invalidRecordThreshold",
        "fileExtension",
        "dataFileURI",
        "charSetConv",
    ]


def test_create_fircosoft_manifest_for_file_raises_when_source_missing():
    with pytest.raises(FileNotFoundError):
        ManifestService.create_fircosoft_manifest_for_file(
            Path("/tmp/does_not_exist_edl.csv"), SOURCE_APPL, SOURCE_FILE_STATIC
        )


def test_create_fircosoft_manifest_for_file_returns_mrk_path(tmp_path):
    csv_path = _csv(tmp_path, ["1,Acme Corp"])

    marker_path = ManifestService.create_fircosoft_manifest_for_file(
        csv_path, SOURCE_APPL, SOURCE_FILE_STATIC
    )

    assert marker_path.name == "Fircosoft_Fcore_EDL_Data_20261008.csv.mrk"

```

#Add edl:

from report_definitions import (
    cander_report,
    china_gtt_report,
    edl_report,
    product_report,
    singapore_report,
    uk_product_report,
)

# ...

# Add to @dataclass class ReportDefinition:

"""
    marker_type ("sha256" default): sidecar-file strategy, see
    reporting_orchestrator.py's _generate_marker_artifact(). Non-"sha256"
    requires both manifest_* fields set.

    output_filename_template (optional): overrides the default
    "{report_name}_{date}.csv" output filename, see _resolve_output_filename()
"""

#...

    # add after sftp_connection: Optional[str] = None
    marker_type: str = "sha256"
    output_filename_template: Optional[str] = None
    manifest_source_appl: Optional[dict] = None
    manifest_source_file_static: Optional[dict] = None

    # replace def __post

        def __post_init__(self):
        if bool(self.sql_file) == bool(self.saved_query_id):
            raise ValueError("Exactly one of sql_file or saved_query_id must be set")
        if self.marker_type != "sha256":
            if not (self.manifest_source_appl and self.manifest_source_file_static):
                raise ValueError(
                    f"marker_type={self.marker_type!r} requires manifest_source_appl "
                    "and manifest_source_file_static to both be set"
                )
            # Pre-declared (any value) so manifest_service.py can update them in
            # place and preserve whichever position they're declared at.
            missing = {
                "recordCount",
                "dataFileURI",
            } - self.manifest_source_file_static.keys()
            if missing:
                raise ValueError(
                    f"manifest_source_file_static is missing {sorted(missing)}"
                )
