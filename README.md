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

app/services/report_definition_service.py:

```python
from dataclasses import dataclass
from pathlib import Path
from typing import Optional

from app.core.config import settings
from app.services.fenergo_service import ReportSource
from report_definitions import (
    cander_report,
    china_gtt_report,
    edl_report,
    product_report,
    singapore_report,
    uk_product_report,
)


@dataclass
class ReportDefinition:
    """Exactly one of sql_file or saved_query_id must be set, never both, never
    neither - mirrors ReportSource's exclusivity rule in fenergo_service.py.

    archive_path (required, e.g. "GTT/China"): where this report's output lives,
    relative to the Archive/Failed root - see reporting_orchestrator.py's
    _resolve_output_directory(). Locally that's settings.download_path/archive_path
    directly (no Archive/Failed split); once settings.AUDIT_ROOT is set (K8s), it
    becomes AUDIT_ROOT/Archive/archive_path on success or AUDIT_ROOT/Failed/
    archive_path if delivery fails after a file was produced (docs/decisions.md
    2026-08-24). Required, not defaulted - every real report needs a real answer
    here, not a silent fallback that could be wrong.

    sftp_connection (optional, e.g. "ClientCentralData"): which named SFTP
    connection (app/services/sftp_connection_service.py) this report delivers to.
    None means "use the legacy single global SFTP_* settings" - see
    delivery_service.py.

    marker_type ("sha256" default): sidecar-file strategy, see
    reporting_orchestrator.py's _generate_marker_artifact(). Non-"sha256"
    requires both manifest_* fields set.

    output_filename_template (optional): overrides the default
    "{report_name}_{date}.csv" output filename, see _resolve_output_filename()."""

    template_file: str
    archive_path: str
    sql_file: Optional[str] = None
    saved_query_id: Optional[str] = None
    generate_marker: bool = True
    sftp_connection: Optional[str] = None
    marker_type: str = "sha256"
    output_filename_template: Optional[str] = None
    manifest_source_appl: Optional[dict] = None
    manifest_source_file_static: Optional[dict] = None

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


class ReportDefinitionService:
    """Owns the report-name -> {sql_file | saved_query_id, template_file} registry."""

    _registry: dict[str, ReportDefinition] = {
        module.REPORT_NAME: ReportDefinition(
            template_file=module.TEMPLATE_FILE,
            archive_path=module.ARCHIVE_PATH,
            sql_file=getattr(module, "SQL_FILE", None),
            saved_query_id=getattr(module, "SAVED_QUERY_ID", None),
            generate_marker=getattr(module, "GENERATE_MARKER", True),
            sftp_connection=getattr(module, "SFTP_CONNECTION", None),
            marker_type=getattr(module, "MARKER_TYPE", "sha256"),
            output_filename_template=getattr(module, "OUTPUT_FILENAME_TEMPLATE", None),
            manifest_source_appl=getattr(module, "MANIFEST_SOURCE_APPL", None),
            manifest_source_file_static=getattr(
                module, "MANIFEST_SOURCE_FILE_STATIC", None
            ),
        )
        for module in (
            cander_report,
            china_gtt_report,
            edl_report,
            product_report,
            singapore_report,
            uk_product_report,
        )
    }

    @classmethod
    def get(cls, report_name: str) -> ReportDefinition:
        try:
            return cls._registry[report_name]
        except KeyError:
            raise KeyError(
                f"No report definition registered for '{report_name}'"
            ) from None

    @classmethod
    def sql_file_path(cls, report_name: str) -> Path:
        definition = cls.get(report_name)
        if definition.sql_file is None:
            raise ValueError(
                f"Report '{report_name}' is saved-query-based (no local .sql file) - "
                f"use report_source() instead of sql_file_path()"
            )
        return settings.sql_query_path / definition.sql_file

    @classmethod
    def template_file_path(cls, report_name: str) -> Path:
        return settings.template_path / cls.get(report_name).template_file

    @classmethod
    def report_source(cls, report_name: str) -> ReportSource:
        """Builds the correct ReportSource for a report - reads the .sql file text
        for sql_file-based definitions, or wraps saved_query_id directly. Callers
        never need to branch on which variant a report definition uses."""
        definition = cls.get(report_name)
        if definition.sql_file is not None:
            return ReportSource(sql_query=cls.sql_file_path(report_name).read_text())
        return ReportSource(saved_query_id=definition.saved_query_id)

```

tests/services/test_report_definition_service.py:

```python
import pytest

from app.core.config import Settings, settings
from app.services.fenergo_service import ReportSource
from app.services.report_definition_service import (
    ReportDefinition,
    ReportDefinitionService,
)


def test_get_returns_known_report_definition():
    definition = ReportDefinitionService.get("CANDERReport")
    assert definition == ReportDefinition(
        sql_file="CanadianDerivatives_OnboardingStatus_YYYYMMDD.sql",
        template_file="CanadianDerivatives_OnboardingStatus.xml",
        archive_path="GTT/Cander",
        sftp_connection="ClientCentralData",
    )
    assert definition.generate_marker is True


def test_generate_marker_defaults_to_true_when_omitted():
    definition = ReportDefinition(
        template_file="x.xml", sql_file="x.sql", archive_path="Test/Path"
    )
    assert definition.generate_marker is True


def test_generate_marker_can_be_explicitly_disabled():
    definition = ReportDefinition(
        template_file="x.xml",
        sql_file="x.sql",
        archive_path="Test/Path",
        generate_marker=False,
    )
    assert definition.generate_marker is False


def test_get_raises_for_unknown_report_name():
    with pytest.raises(KeyError):
        ReportDefinitionService.get("DoesNotExist")


def test_get_returns_china_gtt_report_definition():
    definition = ReportDefinitionService.get("ChinaGTTReport")
    assert definition == ReportDefinition(
        sql_file="China_OnboardingStatus_YYYYMMDD.sql",
        template_file="China_OnboardingStatus.xml",
        archive_path="GTT/China",
        sftp_connection="RegCentral",
    )


def test_get_returns_product_report_definition():
    definition = ReportDefinitionService.get("ProductReport")
    assert definition == ReportDefinition(
        sql_file="Product_OnboardingStatus_YYYYMMDD.sql",
        template_file="Product_OnboardingStatus.xml",
        archive_path="GTT/Product",
        sftp_connection="ClientCentralData",
    )


def test_get_returns_uk_product_report_definition():
    definition = ReportDefinitionService.get("UKProductReport")
    assert definition == ReportDefinition(
        sql_file="UK_Product_OnboardingStatus_YYYYMMDD.sql",
        template_file="UK_Product_OnboardingStatus.xml",
        archive_path="GTT/UKProduct",
        sftp_connection="ClientCentralData",
    )


def test_archive_path_is_required():
    with pytest.raises(TypeError):
        ReportDefinition(template_file="x.xml", sql_file="x.sql")


def test_sftp_connection_defaults_to_none_when_omitted():
    definition = ReportDefinition(
        template_file="x.xml", sql_file="x.sql", archive_path="Test/Path"
    )
    assert definition.sftp_connection is None


def test_sql_and_template_file_paths_resolve_under_settings_paths():
    sql_path = ReportDefinitionService.sql_file_path("CANDERReport")
    template_path = ReportDefinitionService.template_file_path("CANDERReport")

    assert sql_path.name == "CanadianDerivatives_OnboardingStatus_YYYYMMDD.sql"
    assert sql_path.parent == settings.sql_query_path
    assert template_path.name == "CanadianDerivatives_OnboardingStatus.xml"
    assert template_path.parent == settings.template_path


def test_extract_templates_removed_from_settings():
    """Hard rule (CLAUDE.md #5): the registry belongs in report_definitions/ files,
    not hardcoded inside Settings. Settings is for env/secrets only."""
    assert "EXTRACT_TEMPLATES" not in Settings.model_fields


def test_report_definition_rejects_both_sql_file_and_saved_query_id():
    with pytest.raises(ValueError):
        ReportDefinition(
            template_file="x.xml",
            sql_file="x.sql",
            saved_query_id="abc-123",
            archive_path="Test/Path",
        )


def test_report_definition_rejects_neither_sql_file_nor_saved_query_id():
    with pytest.raises(ValueError):
        ReportDefinition(template_file="x.xml", archive_path="Test/Path")


def test_report_source_reads_sql_file_text_for_sql_file_based_report():
    source = ReportDefinitionService.report_source("ChinaGTTReport")

    assert source == ReportSource(
        sql_query=ReportDefinitionService.sql_file_path("ChinaGTTReport").read_text()
    )


def test_report_source_wraps_saved_query_id_for_saved_query_based_report(monkeypatch):
    saved_query_definition = ReportDefinition(
        template_file="China_GTT_Report_ExtractTemplate.xml",
        saved_query_id="saved-query-abc-123",
        archive_path="GTT/China",
    )
    monkeypatch.setitem(
        ReportDefinitionService._registry,
        "SavedQueryTestReport",
        saved_query_definition,
    )

    source = ReportDefinitionService.report_source("SavedQueryTestReport")

    assert source == ReportSource(saved_query_id="saved-query-abc-123")


def test_marker_type_defaults_to_sha256_when_omitted():
    definition = ReportDefinition(
        template_file="x.xml", sql_file="x.sql", archive_path="Test/Path"
    )
    assert definition.marker_type == "sha256"
    assert definition.output_filename_template is None


def test_marker_type_can_be_set_to_fircosoft_manifest():
    definition = ReportDefinition(
        template_file="x.xml",
        sql_file="x.sql",
        archive_path="Test/Path",
        marker_type="fircosoft_manifest",
        output_filename_template="Fircosoft_Fcore_EDL_Data_{date}.csv",
        manifest_source_appl={"providingParty": "gbm"},
        manifest_source_file_static={
            "fileExtension": "csv",
            "recordCount": "",
            "dataFileURI": "",
        },
    )
    assert definition.marker_type == "fircosoft_manifest"
    assert definition.output_filename_template == "Fircosoft_Fcore_EDL_Data_{date}.csv"


def test_non_sha256_marker_type_requires_manifest_fields():
    with pytest.raises(ValueError):
        ReportDefinition(
            template_file="x.xml",
            sql_file="x.sql",
            archive_path="Test/Path",
            marker_type="fircosoft_manifest",
        )


def test_non_sha256_marker_type_requires_record_count_and_data_file_uri_placeholders():
    """recordCount/dataFileURI must be pre-declared keys in
    manifest_source_file_static (any value) so manifest_service.py can update
    them in place and preserve whatever position they're declared at."""
    with pytest.raises(ValueError):
        ReportDefinition(
            template_file="x.xml",
            sql_file="x.sql",
            archive_path="Test/Path",
            marker_type="fircosoft_manifest",
            manifest_source_appl={"providingParty": "gbm"},
            manifest_source_file_static={"fileExtension": "csv"},
        )


def test_get_returns_edl_report_definition():
    definition = ReportDefinitionService.get("EDLReport")
    assert definition.archive_path == "EDL"
    assert definition.sftp_connection == "ClientCentralData"
    assert definition.marker_type == "fircosoft_manifest"
    assert definition.output_filename_template == "Fircosoft_Fcore_EDL_Data_{date}.csv"
    assert definition.manifest_source_appl["appAcronym"] == "b8fb"
    assert definition.manifest_source_file_static["fileExtension"] == "csv"


def test_sql_file_path_raises_clearly_for_saved_query_based_report(monkeypatch):
    saved_query_definition = ReportDefinition(
        template_file="China_GTT_Report_ExtractTemplate.xml",
        saved_query_id="saved-query-abc-123",
        archive_path="GTT/China",
    )
    monkeypatch.setitem(
        ReportDefinitionService._registry,
        "SavedQueryTestReport",
        saved_query_definition,
    )

    with pytest.raises(ValueError):
        ReportDefinitionService.sql_file_path("SavedQueryTestReport")

```

app/services/reporting_orchestrator.py:

```python
import shutil
import uuid
from dataclasses import dataclass
from datetime import datetime, timezone
from pathlib import Path
from typing import Optional

from app.core.config import settings
from app.core.logging import log
from app.models.execution import (
    ExecutionOperationType,
    ExecutionStatus,
    FileProcessingStatus,
    FileProcessingStep,
    ReportExecution,
)
from app.services import execution_service
from app.services.delivery_service import DeliveryConfigError, DeliveryService
from app.services.download_service import DownloadService
from app.services.fenergo_service import FenergoService
from app.services.manifest_service import ManifestService
from app.services.marker_service import MarkerService
from app.services.polling_service import PollResult, poll_until_ready
from app.services.report_definition_service import (
    ReportDefinition,
    ReportDefinitionService,
)
from app.services.sftp_connection_service import SFTPConnectionService
from app.services.transform.template_reader import TemplateReader
from app.services.transform.transformation_service import TransformationService


class OrchestrationError(Exception):
    """Raised when polling finishes without reaching Completed - the failure is
    already recorded via execution_service.mark_failed before this is raised."""


@dataclass
class UtilityOperationResult:
    """Result of transform_existing_csv()/generate_marker_for_file()/
    deliver_file() (Round 2, docs/schema_alignment.md). execution_id is None
    when the caller gave no --execution-id - the operation still ran, but
    nothing was persisted to the DB: report_execution rows are only ever
    created by a real Fenergo submission now, so there's no valid parent for
    a file_processing row to attach to when the caller has no execution to
    link against."""

    execution_id: Optional[str]
    output_path: str


def _record_file_processing_safely(
    execution_id: str, processing_step: str, status: str, **kwargs
) -> None:
    """Used only by the FULL_RUN pipeline (submit_report/poll_execution/
    download_execution/transform_execution/run_report's own block), where
    execution_service.mark_*/mark_failed is already the authoritative outcome -
    this write is purely supplementary observability. A failure here must never
    retroactively flip a genuinely successful stage to FAILED (for SUBMIT
    specifically, that would risk a caller retrying into a duplicate Fenergo
    submission - docs/decisions.md 2026-08-04) or mask the real original
    exception. Contrast the utility flows (transform_existing_csv/
    generate_marker_for_file/deliver_file), which call
    execution_service.record_file_processing() directly - there, it's the sole
    authoritative record of the operation's outcome, so a write failure should
    propagate like any other real failure."""
    try:
        execution_service.record_file_processing(
            execution_id, processing_step, status, **kwargs
        )
    except Exception as exc:
        log.error(
            f"record_file_processing failed for {execution_id} "
            f"step={processing_step} status={status}: {exc}"
        )


def _validate_delivery_target(
    sftp: Optional[bool], local_destination: Optional[Path]
) -> None:
    """Not all workflows require SFTP (net new, 2026-08-13) - run_report()/
    transform_existing_csv()/generate_marker_for_file() all take an optional
    sftp flag (tri-state, same None/True/False pattern as generate_marker - None
    means "no explicit override, use the default") and an optional
    local_destination as an alternative delivery target. Explicitly requesting
    both is a usage error, not a silently-resolved precedence rule - the caller
    must pick one."""
    if local_destination is not None and sftp is True:
        raise ValueError(
            "Choose one delivery target: --sftp or --output-path, not both"
        )


def _resolve_output_directory(archive_path: str, failed: bool = False) -> Path:
    """Where a report's transformed output (and marker) actually gets written -
    net new 2026-08-24, see ReportDefinition.archive_path's docstring. Locally
    (AUDIT_ROOT unset): settings.download_path/<archive_path>/, no Archive/Failed
    split at all - `failed` has no effect. In K8s (AUDIT_ROOT set to the real PVC
    mount, e.g. "/app/Audit"): AUDIT_ROOT/Archive/<archive_path>/ on success,
    AUDIT_ROOT/Failed/<archive_path>/ once a delivery failure relocates it there."""
    if settings.AUDIT_ROOT:
        subtree = "Failed" if failed else "Archive"
        return Path(settings.AUDIT_ROOT) / subtree / archive_path
    return settings.download_path / archive_path


def _resolve_output_filename(
    report_name: str, definition: ReportDefinition, timestamp: str
) -> str:
    # output_filename_template overrides the default naming - see EDLReport.
    if definition.output_filename_template:
        return definition.output_filename_template.format(date=timestamp)
    return f"{report_name}_{timestamp}.csv"


def _move_to_failed_on_post_transform_failure(
    output_path: Path, marker_path: Optional[Path], archive_path: str, stage: str
) -> None:
    """Only when a real file was already produced and something after it failed
    (MARKER or DELIVER stage - never SUBMIT/POLL/DOWNLOAD/TRANSFORM, which have no
    file yet) - relocates it from its Archive/ location to Failed/<archive_path>
    (docs/decisions.md 2026-08-24). Best-effort: a failure here must never mask
    the real original exception, which is why callers wrap this and always
    re-raise regardless of what happens in here."""
    if stage not in ("MARKER", "DELIVER"):
        return
    try:
        failed_dir = _resolve_output_directory(archive_path, failed=True)
        failed_dir.mkdir(parents=True, exist_ok=True)
        if output_path.exists():
            shutil.move(str(output_path), str(failed_dir / output_path.name))
        if marker_path is not None and marker_path.exists():
            shutil.move(str(marker_path), str(failed_dir / marker_path.name))
    except Exception as move_exc:
        log.error(f"Failed to move output to Failed/ location: {move_exc}")


def _copy_to_local_path(local_path: Path, output_directory: Path) -> Path:
    """The --output-path alternative to SFTP delivery - a plain filesystem copy,
    deliberately not added to delivery_service.py (that module's scope is SFTP
    only, per its own docstring and CLAUDE.md's service-boundary table)."""
    output_directory = Path(output_directory)
    output_directory.mkdir(parents=True, exist_ok=True)
    destination = output_directory / local_path.name
    shutil.copy2(local_path, destination)
    return destination


class ReportingOrchestrator:
    """Sequences fenergo_service -> polling_service -> download_service ->
    transform/ -> marker_service -> delivery_service into Flow 1/2/3
    (docs/architecture.md Sec 4), tracking lifecycle via execution_service at each
    step. No business logic of its own beyond sequencing + logging - see CLAUDE.md
    service boundaries.

    Marker generation (a .mrk SHA256 checksum file alongside the transformed CSV)
    is fatal on failure, same as every other stage - a bad output file or a
    disk/permissions problem is worth surfacing loudly, not swallowing (see
    docs/decisions.md 2026-07-31). Whether a given run generates one at all is
    resolved per-call: an explicit generate_marker argument wins if given,
    otherwise the report's own ReportDefinition.generate_marker default applies
    (docs/decisions.md). When a marker is generated, both the CSV and its .mrk
    file are delivered over SFTP so a downstream consumer can verify integrity
    themselves.

    run_report() (Flow 1) creates a report_execution row via a real Fenergo
    submission - the only way one is ever created (Round 2, docs/schema_alignment.md).
    transform_existing_csv() (Flow 2), generate_marker_for_file() (Flow 3), and
    deliver_file() are "utility" operations with no Fenergo call of their own -
    each takes an optional execution_id: given, real file_processing rows are
    written under that existing execution (never mutating the execution itself);
    omitted, the operation runs untracked (log-only). See UtilityOperationResult.

    submit_report()/poll_execution()/download_execution()/transform_execution()
    are the same submit->poll->download->transform sequence broken into
    independently-callable, resumable stages - each takes/returns an existing
    execution_id and is responsible for its own mark_failed on error, so each can
    be invoked standalone (as its own Airflow task/CLI command) or chained by
    run_report() below, which now just calls them in order. This is what lets a
    retry of a failed poll/download/transform task resume the same execution_id
    instead of resubmitting to Fenergo from scratch - see docs/decisions.md
    2026-08-04 for why this only narrows, not closes, that risk.
    """

    def __init__(
        self,
        fenergo_service: Optional[FenergoService] = None,
        download_service: Optional[DownloadService] = None,
        delivery_service: Optional[DeliveryService] = None,
        template_reader=TemplateReader,
        transformation_service: Optional[TransformationService] = None,
        marker_service=MarkerService,
        manifest_service=ManifestService,
    ):
        self.fenergo_service = fenergo_service or FenergoService()
        self.download_service = download_service or DownloadService()
        self.delivery_service = delivery_service or DeliveryService()
        self.template_reader = template_reader
        self.transformation_service = transformation_service or TransformationService()
        self.marker_service = marker_service
        self.manifest_service = manifest_service

    def _generate_marker_artifact(
        self, output_path: Path, definition: ReportDefinition
    ) -> Path:
        # Dispatches on marker_type - "sha256" (default) hashes the file; any
        # other value builds a manifest instead, see manifest_service.py.
        if definition.marker_type == "fircosoft_manifest":
            return self.manifest_service.create_fircosoft_manifest_for_file(
                output_path,
                definition.manifest_source_appl,
                definition.manifest_source_file_static,
            )
        return self.marker_service.create_marker_for_file(output_path)

    async def submit_report(self, report_name: str) -> ReportExecution:
        """Stage 1, standalone-callable: submit report_name to Fenergo and persist
        the resulting fenergo_report_id. Does not poll/download/transform/deliver -
        see poll_execution()/download_execution()/transform_execution().

        Resumes an existing in-progress execution instead of resubmitting, if one
        exists (docs/decisions.md) - avoids duplicate Fenergo submissions on retry."""
        existing = execution_service.get_in_progress_execution(report_name)
        if existing is not None:
            log.info(
                f"submit_report: resuming in-progress execution "
                f"{existing.execution_id} for {report_name} instead of resubmitting"
            )
            return existing

        execution = execution_service.create_execution(
            report_name, operation_type=ExecutionOperationType.FULL_RUN.value
        )
        stage_started = datetime.now(timezone.utc)
        try:
            source = ReportDefinitionService.report_source(report_name)
            submit_result = await self.fenergo_service.submit(
                source=source, description=f"{report_name} orchestrated run"
            )
            updated = execution_service.mark_submitted(
                execution.execution_id, submit_result.report_id
            )
            _record_file_processing_safely(
                execution.execution_id,
                FileProcessingStep.SUBMIT.value,
                FileProcessingStatus.COMPLETED.value,
                started_utc=stage_started,
                completed_utc=datetime.now(timezone.utc),
            )
            return updated
        except Exception as exc:
            log.error(f"submit_report failed for {report_name}: {exc}")
            execution_service.mark_failed(execution.execution_id, "SUBMIT", str(exc))
            _record_file_processing_safely(
                execution.execution_id,
                FileProcessingStep.SUBMIT.value,
                FileProcessingStatus.FAILED.value,
                started_utc=stage_started,
                completed_utc=datetime.now(timezone.utc),
                error_message=str(exc),
            )
            raise

    async def poll_execution(self, execution_id: str, once: bool = False) -> PollResult:
        """Stage 2, standalone-callable: poll Fenergo for an execution
        submit_report() already created. Returns PollResult (not ReportExecution) -
        presigned_url is deliberately not persisted (ephemeral, expires), so the
        caller/Airflow must carry it forward directly to download_execution().

        once=False (default): loops internally via poll_until_ready() until
        Completed/Failed/TimedOut, same as always - can block for
        POLLING_TIMEOUT_MINUTES.

        once=True: a single check_status() call, no internal wait-loop. If still
        in progress (not Completed, not Failed), returns normally with that raw
        status - does NOT raise, does NOT mark_failed. "Not done yet" is not a
        failure and must never be written to the DB as one. This is what lets
        `poll --once` be driven as a sensor via Airflow's own task
        retries/retry_delay instead of blocking a worker slot - see
        docs/decisions.md 2026-08-04.
        """
        # get_execution() runs outside the try below deliberately - a bad/unknown
        # execution_id is a usage error that should raise LookupError directly,
        # not be caught and re-marked-failed against a row that doesn't exist.
        execution = execution_service.get_execution(execution_id)
        try:
            stage_started = datetime.now(timezone.utc)
            if not execution.fenergo_report_id:
                raise ValueError(
                    f"execution {execution_id} has no fenergo_report_id - "
                    "run submit_report()/`submit` first"
                )

            if once:
                # Counts every sensor-style poke, not just loop-mode attempts -
                # existing field, previously unused by any caller.
                execution_service.increment_poll_attempt(execution_id)
                status_result = await self.fenergo_service.check_status(
                    execution.fenergo_report_id
                )
                if status_result.status == "Completed":
                    execution_service.mark_url_received(
                        execution_id, presigned_url_expiry=status_result.expiration
                    )
                    _record_file_processing_safely(
                        execution_id,
                        FileProcessingStep.POLL.value,
                        FileProcessingStatus.COMPLETED.value,
                        started_utc=stage_started,
                        completed_utc=datetime.now(timezone.utc),
                    )
                elif status_result.status == "Failed":
                    raise OrchestrationError(
                        f"Fenergo reported Failed for execution {execution_id}"
                    )
                return PollResult(
                    status=status_result.status,
                    presigned_url=status_result.presigned_url,
                    presigned_url_expiration=status_result.expiration,
                )

            poll_result = await poll_until_ready(
                self.fenergo_service, execution.fenergo_report_id
            )
            if poll_result.status != "Completed":
                raise OrchestrationError(
                    f"Polling did not complete for execution {execution_id}: "
                    f"status={poll_result.status} error={poll_result.error}"
                )
            execution_service.mark_url_received(
                execution_id, presigned_url_expiry=poll_result.presigned_url_expiration
            )
            _record_file_processing_safely(
                execution_id,
                FileProcessingStep.POLL.value,
                FileProcessingStatus.COMPLETED.value,
                started_utc=stage_started,
                completed_utc=datetime.now(timezone.utc),
            )
            return poll_result
        except Exception as exc:
            log.error(f"poll_execution failed for {execution_id}: {exc}")
            execution_service.mark_failed(execution_id, "POLL", str(exc))
            if execution.fenergo_report_id:
                # Excludes the pre-Fenergo-call ValueError branch above - no
                # real poll was ever attempted, so no step to record.
                _record_file_processing_safely(
                    execution_id,
                    FileProcessingStep.POLL.value,
                    FileProcessingStatus.FAILED.value,
                    started_utc=stage_started,
                    completed_utc=datetime.now(timezone.utc),
                    error_message=str(exc),
                )
            raise

    async def download_execution(
        self, execution_id: str, presigned_url: str
    ) -> ReportExecution:
        """Stage 3, standalone-callable: download the CSV poll_execution() made
        ready, for an execution submit_report() already created."""
        execution = execution_service.get_execution(execution_id)
        stage_started = datetime.now(timezone.utc)
        try:
            if not execution.fenergo_report_id:
                raise ValueError(
                    f"execution {execution_id} has no fenergo_report_id - "
                    "run submit_report()/`submit` first"
                )
            download_result = await self.download_service.download(
                presigned_url=presigned_url,
                destination_filename=f"{execution.fenergo_report_id}.csv",
            )
            updated = execution_service.mark_downloaded(
                execution_id, str(download_result.local_path)
            )
            _record_file_processing_safely(
                execution_id,
                FileProcessingStep.DOWNLOAD.value,
                FileProcessingStatus.COMPLETED.value,
                started_utc=stage_started,
                completed_utc=datetime.now(timezone.utc),
                file_name=download_result.local_path.name,
                output_file_path=str(download_result.local_path),
            )
            return updated
        except Exception as exc:
            log.error(f"download_execution failed for {execution_id}: {exc}")
            execution_service.mark_failed(execution_id, "DOWNLOAD", str(exc))
            if execution.fenergo_report_id:
                _record_file_processing_safely(
                    execution_id,
                    FileProcessingStep.DOWNLOAD.value,
                    FileProcessingStatus.FAILED.value,
                    started_utc=stage_started,
                    completed_utc=datetime.now(timezone.utc),
                    error_message=str(exc),
                )
            raise

    async def transform_execution(self, execution_id: str) -> ReportExecution:
        """Stage 4, standalone-callable: transform the CSV download_execution()
        produced, against the execution's report_name's template. Persists the
        transformed output path onto the row immediately (mark_transformed(
        output_file_path=...)) rather than waiting for a final mark_completed,
        since "final" may now be a separate deliver invocation. Only ever called
        against a real FULL_RUN execution - run_report()'s chain or the
        standalone `transform` CLI stage command - transform_existing_csv()
        (Flow 2) no longer calls this method, see its own docstring."""
        execution = execution_service.get_execution(execution_id)
        stage_started = datetime.now(timezone.utc)
        try:
            if not execution.report_name:
                raise ValueError(
                    f"execution {execution_id} has no report_name - transform "
                    "needs a registered report's template"
                )
            if not execution.downloaded_file_path:
                raise ValueError(
                    f"execution {execution_id} has no downloaded_file_path - "
                    "run download_execution()/`download` first"
                )
            definition = ReportDefinitionService.get(execution.report_name)
            schema = self.template_reader.parse_template(
                ReportDefinitionService.template_file_path(execution.report_name)
            )
            output_dir = _resolve_output_directory(definition.archive_path)
            output_dir.mkdir(parents=True, exist_ok=True)
            output_path = output_dir / _resolve_output_filename(
                execution.report_name, definition, self._timestamp()
            )
            self.transformation_service.transform_file(
                input_path=Path(execution.downloaded_file_path),
                output_path=output_path,
                schema=schema,
            )
            updated = execution_service.mark_transformed(
                execution_id, output_file_path=str(output_path)
            )
            _record_file_processing_safely(
                execution_id,
                FileProcessingStep.TRANSFORM.value,
                FileProcessingStatus.COMPLETED.value,
                started_utc=stage_started,
                completed_utc=datetime.now(timezone.utc),
                file_name=output_path.name,
                output_file_path=str(output_path),
            )
            return updated
        except Exception as exc:
            log.error(f"transform_execution failed for {execution_id}: {exc}")
            execution_service.mark_failed(execution_id, "TRANSFORM", str(exc))
            _record_file_processing_safely(
                execution_id,
                FileProcessingStep.TRANSFORM.value,
                FileProcessingStatus.FAILED.value,
                started_utc=stage_started,
                completed_utc=datetime.now(timezone.utc),
                error_message=str(exc),
            )
            raise

    def deliver_file(
        self,
        local_path: Path,
        target: str = "success",
        remote_filename: Optional[str] = None,
        execution_id: Optional[str] = None,
    ) -> UtilityOperationResult:
        """Deliver an arbitrary local file over SFTP - no report or template
        involved. Promotes what poc/diagnostic_deliver.py already did manually
        into a real, tracked-when-linked production command. Tracking is opt-in
        via execution_id (Round 2, docs/schema_alignment.md) - see
        UtilityOperationResult."""
        if execution_id is not None:
            execution_service.get_execution(execution_id)  # raises if unknown
        stage_started = datetime.now(timezone.utc)
        try:
            remote_directory = self._resolve_target_directory(target)
            delivery_result = self.delivery_service.deliver(
                local_path=local_path,
                remote_directory=remote_directory,
                remote_filename=remote_filename,
            )
            if execution_id is not None:
                execution_service.record_file_processing(
                    execution_id,
                    FileProcessingStep.DELIVER.value,
                    FileProcessingStatus.COMPLETED.value,
                    started_utc=stage_started,
                    completed_utc=datetime.now(timezone.utc),
                    file_name=local_path.name,
                    output_file_path=delivery_result.remote_path,
                )
            else:
                log.info(
                    f"deliver_file completed untracked (no execution_id): {local_path}"
                )
            return UtilityOperationResult(
                execution_id=execution_id, output_path=delivery_result.remote_path
            )
        except Exception as exc:
            log.error(f"deliver_file failed for {local_path}: {exc}")
            if execution_id is not None:
                execution_service.record_file_processing(
                    execution_id,
                    FileProcessingStep.DELIVER.value,
                    FileProcessingStatus.FAILED.value,
                    started_utc=stage_started,
                    completed_utc=datetime.now(timezone.utc),
                    error_message=str(exc),
                )
            raise

    @staticmethod
    def _resolve_target_directory(target: str) -> str:
        if target == "success":
            directory = settings.SFTP_SUCCESS_DIRECTORY
            setting_name = "SFTP_SUCCESS_DIRECTORY"
        elif target == "error":
            directory = settings.SFTP_ERROR_DIRECTORY
            setting_name = "SFTP_ERROR_DIRECTORY"
        else:
            raise ValueError(
                f"Unknown delivery target {target!r} - use 'success' or 'error'"
            )
        if not directory:
            raise DeliveryConfigError(
                f"{setting_name} must be set to deliver to {target!r}"
            )
        return directory

    def _resolve_delivery(
        self,
        local_path: Path,
        sftp: Optional[bool],
        local_destination: Optional[Path],
        sftp_connection_name: Optional[str] = None,
    ) -> Optional[str]:
        """The single decision point for where an artifact (transformed CSV,
        marker file) ends up - net new 2026-08-13, since not all workflows
        require SFTP. Returns the resulting location (SFTP remote path or local
        copy destination) for file_processing tracking, or None if delivery was
        skipped entirely (sftp=False, no local_destination given) - callers
        should treat None as "nothing happened, don't record a DELIVER step."
        _validate_delivery_target() must already have been called by this
        point - this method doesn't re-check the both-given usage error.

        sftp_connection_name (net new 2026-08-24, from ReportDefinition.
        sftp_connection): when set, resolves that named connection
        (sftp_connection_service.py) and delivers to *its* remote_folder -
        replacing the legacy global SFTP_SUCCESS_DIRECTORY for that call
        entirely, not falling back to it. None (the default - used by
        generate_marker_for_file(), which has no report/connection context)
        keeps the exact legacy single-connection behavior unchanged."""
        if local_destination is not None:
            return str(_copy_to_local_path(local_path, local_destination))
        if sftp is False:
            return None

        if sftp_connection_name:
            connection = SFTPConnectionService.get(sftp_connection_name)
            remote_directory = connection.remote_folder
        else:
            connection = None
            if not settings.SFTP_SUCCESS_DIRECTORY:
                raise DeliveryConfigError(
                    "SFTP_SUCCESS_DIRECTORY must be set to deliver via SFTP - "
                    "or pass --output-path to copy locally, or --no-sftp to skip "
                    "delivery entirely"
                )
            remote_directory = settings.SFTP_SUCCESS_DIRECTORY

        delivery_result = self.delivery_service.deliver(
            local_path=local_path,
            remote_directory=remote_directory,
            connection=connection,
        )
        return delivery_result.remote_path

    async def run_report(
        self,
        report_name: str,
        generate_marker: Optional[bool] = None,
        sftp: Optional[bool] = None,
        local_destination: Optional[Path] = None,
        track: bool = True,
    ) -> ReportExecution:
        """sftp/local_destination (net new, 2026-08-13): not all workflows
        require SFTP delivery. sftp=None/True (default): deliver via the
        configured SFTP landing zone, same as always. sftp=False with no
        local_destination: skip delivery entirely - the row stays at
        TRANSFORMED, mark_completed() is never called, since nothing was
        actually delivered anywhere. local_destination given: copy the
        output (and marker, if generated) there instead of SFTP -
        mark_completed() IS still called in this case (a real delivery
        happened, just not via SFTP) - see _resolve_delivery().

        track=False (2026-09-16): same real pipeline, no execution_service
        calls at all - for Airflow smoke-testing without writing rows. No
        idempotency guard in this mode - never enable retries on it."""
        _validate_delivery_target(sftp, local_destination)
        if not track:
            return await self._run_report_untracked(
                report_name, generate_marker, sftp, local_destination
            )
        execution = await self.submit_report(report_name)
        poll_result = await self.poll_execution(execution.execution_id)
        execution = await self.download_execution(
            execution.execution_id, poll_result.presigned_url
        )
        execution = await self.transform_execution(execution.execution_id)

        definition = ReportDefinitionService.get(report_name)
        effective_generate_marker = (
            generate_marker
            if generate_marker is not None
            else definition.generate_marker
        )
        output_path = Path(execution.output_file_path)

        stage = "MARKER"
        stage_started = datetime.now(timezone.utc)
        try:
            marker_path = None
            if effective_generate_marker:
                marker_path = self._generate_marker_artifact(output_path, definition)
                _record_file_processing_safely(
                    execution.execution_id,
                    FileProcessingStep.MARKER.value,
                    FileProcessingStatus.COMPLETED.value,
                    started_utc=stage_started,
                    completed_utc=datetime.now(timezone.utc),
                    file_name=output_path.name,
                    output_file_path=str(marker_path),
                    checksum_value=(
                        marker_path.read_text().strip()
                        if definition.marker_type == "sha256"
                        else None
                    ),
                )

            stage = "DELIVER"
            stage_started = datetime.now(timezone.utc)
            delivered_location = self._resolve_delivery(
                output_path, sftp, local_destination, definition.sftp_connection
            )
            if delivered_location is not None:
                _record_file_processing_safely(
                    execution.execution_id,
                    FileProcessingStep.DELIVER.value,
                    FileProcessingStatus.COMPLETED.value,
                    started_utc=stage_started,
                    completed_utc=datetime.now(timezone.utc),
                    file_name=output_path.name,
                    output_file_path=delivered_location,
                )
            if marker_path is not None:
                stage_started = datetime.now(timezone.utc)
                marker_delivered_location = self._resolve_delivery(
                    marker_path, sftp, local_destination, definition.sftp_connection
                )
                if marker_delivered_location is not None:
                    _record_file_processing_safely(
                        execution.execution_id,
                        FileProcessingStep.DELIVER.value,
                        FileProcessingStatus.COMPLETED.value,
                        started_utc=stage_started,
                        completed_utc=datetime.now(timezone.utc),
                        file_name=marker_path.name,
                        output_file_path=marker_delivered_location,
                    )

            if delivered_location is None:
                # Delivery skipped entirely (sftp=False, no local_destination) -
                # `execution` already reflects TRANSFORMED from the chained
                # transform_execution() call above, nothing more to persist.
                return execution
            return execution_service.mark_completed(
                execution.execution_id, str(output_path)
            )

        except Exception as exc:
            log.error(f"Orchestration failed for {report_name} at stage={stage}: {exc}")
            execution_service.mark_failed(execution.execution_id, stage, str(exc))
            _move_to_failed_on_post_transform_failure(
                output_path, marker_path, definition.archive_path, stage
            )
            _record_file_processing_safely(
                execution.execution_id,
                stage,
                FileProcessingStatus.FAILED.value,
                started_utc=stage_started,
                completed_utc=datetime.now(timezone.utc),
                error_message=str(exc),
            )
            raise

    async def _run_report_untracked(
        self,
        report_name: str,
        generate_marker: Optional[bool],
        sftp: Optional[bool],
        local_destination: Optional[Path],
    ) -> ReportExecution:
        """track=False path - doesn't call submit_report()/poll_execution()/
        download_execution()/transform_execution(), since those read/write a
        persisted execution_id. State is carried via a local, never-persisted
        ReportExecution instead, so the return contract matches the tracked
        path."""
        execution = ReportExecution(
            execution_id=str(uuid.uuid4()),
            report_name=report_name,
            status=ExecutionStatus.CREATED.value,
        )
        stage = "SUBMIT"
        definition = None
        output_path = None
        marker_path = None
        try:
            source = ReportDefinitionService.report_source(report_name)
            submit_result = await self.fenergo_service.submit(
                source=source,
                description=f"{report_name} orchestrated run (untracked)",
            )
            execution.fenergo_report_id = submit_result.report_id
            execution.status = ExecutionStatus.REQUEST_SUBMITTED.value

            stage = "POLL"
            poll_result = await poll_until_ready(
                self.fenergo_service, submit_result.report_id
            )
            if poll_result.status != "Completed":
                raise OrchestrationError(
                    f"Polling did not complete for {report_name} (untracked): "
                    f"status={poll_result.status} error={poll_result.error}"
                )
            execution.status = ExecutionStatus.REPORT_READY.value

            stage = "DOWNLOAD"
            download_result = await self.download_service.download(
                presigned_url=poll_result.presigned_url,
                destination_filename=f"{submit_result.report_id}.csv",
            )
            execution.downloaded_file_path = str(download_result.local_path)
            execution.status = ExecutionStatus.DOWNLOADED.value

            stage = "TRANSFORM"
            definition = ReportDefinitionService.get(report_name)
            schema = self.template_reader.parse_template(
                ReportDefinitionService.template_file_path(report_name)
            )
            output_dir = _resolve_output_directory(definition.archive_path)
            output_dir.mkdir(parents=True, exist_ok=True)
            output_path = output_dir / _resolve_output_filename(
                report_name, definition, self._timestamp()
            )
            self.transformation_service.transform_file(
                input_path=Path(execution.downloaded_file_path),
                output_path=output_path,
                schema=schema,
            )
            execution.output_file_path = str(output_path)
            execution.status = ExecutionStatus.TRANSFORMED.value

            effective_generate_marker = (
                generate_marker
                if generate_marker is not None
                else definition.generate_marker
            )

            stage = "MARKER"
            if effective_generate_marker:
                marker_path = self._generate_marker_artifact(output_path, definition)

            stage = "DELIVER"
            delivered_location = self._resolve_delivery(
                output_path, sftp, local_destination, definition.sftp_connection
            )
            if marker_path is not None:
                self._resolve_delivery(
                    marker_path, sftp, local_destination, definition.sftp_connection
                )

            if delivered_location is not None:
                execution.status = ExecutionStatus.SFTP_COMPLETED.value
            log.info(f"run_report completed untracked (--no-track): {report_name}")
            return execution

        except Exception as exc:
            log.error(
                f"Untracked orchestration failed for {report_name} "
                f"at stage={stage}: {exc}"
            )
            execution.status = ExecutionStatus.FAILED.value
            execution.failure_stage = stage
            execution.error_message = str(exc)
            if definition is not None:
                _move_to_failed_on_post_transform_failure(
                    output_path, marker_path, definition.archive_path, stage
                )
            raise

    async def transform_existing_csv(
        self,
        report_name: str,
        input_csv_path: Path,
        generate_marker: Optional[bool] = None,
        execution_id: Optional[str] = None,
        sftp: Optional[bool] = None,
        local_destination: Optional[Path] = None,
    ) -> UtilityOperationResult:
        """Flow 2 (docs/architecture.md Sec 4): transform an already-downloaded
        CSV against report_name's template, without submitting to or polling
        Fenergo. Tracking is opt-in via execution_id (Round 2,
        docs/schema_alignment.md): given, must reference an existing execution
        for this same report_name (a prior FULL_RUN) - writes real
        file_processing rows under it, but deliberately never calls
        transform_execution()/mutates that execution's own report_execution row
        (re-processing is a file-level event, not a change to the original run's
        outcome). Omitted: untracked, log-only - see UtilityOperationResult.

        sftp/local_destination (net new, 2026-08-13): not all workflows
        require SFTP - see run_report()'s docstring for the full sftp=None/
        True/False + local_destination decision matrix, same here."""
        _validate_delivery_target(sftp, local_destination)
        if execution_id is not None:
            execution = execution_service.get_execution(execution_id)
            if execution.report_name != report_name:
                raise ValueError(
                    f"execution {execution_id} belongs to report "
                    f"'{execution.report_name}', not '{report_name}'"
                )

        stage = "TRANSFORM"
        stage_started = datetime.now(timezone.utc)
        # Predefined here (not just inside the try below) so the except block can
        # safely reference them for Failed/-routing even if an exception happens
        # before they'd otherwise be assigned - only actually used when
        # definition is not None (stage reached MARKER/DELIVER for real).
        definition = None
        output_path = None
        marker_path = None
        try:
            definition = ReportDefinitionService.get(report_name)
            effective_generate_marker = (
                generate_marker
                if generate_marker is not None
                else definition.generate_marker
            )

            schema = self.template_reader.parse_template(
                ReportDefinitionService.template_file_path(report_name)
            )
            output_dir = _resolve_output_directory(definition.archive_path)
            output_dir.mkdir(parents=True, exist_ok=True)
            output_path = output_dir / _resolve_output_filename(
                report_name, definition, self._timestamp()
            )
            self.transformation_service.transform_file(
                input_path=Path(input_csv_path),
                output_path=output_path,
                schema=schema,
            )
            if execution_id is not None:
                execution_service.record_file_processing(
                    execution_id,
                    FileProcessingStep.TRANSFORM.value,
                    FileProcessingStatus.COMPLETED.value,
                    started_utc=stage_started,
                    file_name=output_path.name,
                    output_file_path=str(output_path),
                )

            stage = "MARKER"
            stage_started = datetime.now(timezone.utc)
            marker_path = None
            if effective_generate_marker:
                marker_path = self._generate_marker_artifact(output_path, definition)
                if execution_id is not None:
                    execution_service.record_file_processing(
                        execution_id,
                        FileProcessingStep.MARKER.value,
                        FileProcessingStatus.COMPLETED.value,
                        started_utc=stage_started,
                        file_name=marker_path.name,
                        output_file_path=str(marker_path),
                        checksum_value=(
                            marker_path.read_text().strip()
                            if definition.marker_type == "sha256"
                            else None
                        ),
                    )

            stage = "DELIVER"
            stage_started = datetime.now(timezone.utc)
            delivered_location = self._resolve_delivery(
                output_path, sftp, local_destination, definition.sftp_connection
            )
            if execution_id is not None and delivered_location is not None:
                execution_service.record_file_processing(
                    execution_id,
                    FileProcessingStep.DELIVER.value,
                    FileProcessingStatus.COMPLETED.value,
                    started_utc=stage_started,
                    file_name=output_path.name,
                    output_file_path=delivered_location,
                )
            if marker_path is not None:
                stage_started = datetime.now(timezone.utc)
                marker_delivered_location = self._resolve_delivery(
                    marker_path, sftp, local_destination, definition.sftp_connection
                )
                if execution_id is not None and marker_delivered_location is not None:
                    execution_service.record_file_processing(
                        execution_id,
                        FileProcessingStep.DELIVER.value,
                        FileProcessingStatus.COMPLETED.value,
                        started_utc=stage_started,
                        file_name=marker_path.name,
                        output_file_path=marker_delivered_location,
                    )

            if execution_id is None:
                log.info(
                    f"transform_existing_csv completed untracked (no execution_id): "
                    f"{input_csv_path}"
                )
            return UtilityOperationResult(
                execution_id=execution_id, output_path=str(output_path)
            )

        except Exception as exc:
            log.error(
                f"Transform-only orchestration failed for {report_name} "
                f"at stage={stage}: {exc}"
            )
            if definition is not None:
                _move_to_failed_on_post_transform_failure(
                    output_path, marker_path, definition.archive_path, stage
                )
            if execution_id is not None:
                execution_service.record_file_processing(
                    execution_id,
                    stage,
                    FileProcessingStatus.FAILED.value,
                    started_utc=stage_started,
                    error_message=str(exc),
                )
            raise

    def generate_marker_for_file(
        self,
        file_path: Path,
        execution_id: Optional[str] = None,
        sftp: Optional[bool] = None,
        local_destination: Optional[Path] = None,
    ) -> UtilityOperationResult:
        """Flow 3 (docs/architecture.md Sec 4): generate and deliver a .mrk
        checksum for an arbitrary file - no report or template involved.
        Tracking is opt-in via execution_id (Round 2, docs/schema_alignment.md) -
        see deliver_file()'s docstring for the same linked/untracked split.

        sftp/local_destination (net new, 2026-08-13): not all workflows
        require SFTP - see run_report()'s docstring for the full sftp=None/
        True/False + local_destination decision matrix, same here. Unlike
        run_report()/transform_existing_csv(), there's no "stays at TRANSFORMED"
        terminal-status concern here - this flow never creates/mutates a
        ReportExecution row regardless of delivery outcome."""
        _validate_delivery_target(sftp, local_destination)
        if execution_id is not None:
            execution_service.get_execution(execution_id)  # raises if unknown
        stage = "MARKER"
        stage_started = datetime.now(timezone.utc)
        try:
            marker_path = self.marker_service.create_marker_for_file(file_path)
            if execution_id is not None:
                execution_service.record_file_processing(
                    execution_id,
                    FileProcessingStep.MARKER.value,
                    FileProcessingStatus.COMPLETED.value,
                    started_utc=stage_started,
                    file_name=marker_path.name,
                    output_file_path=str(marker_path),
                    checksum_value=marker_path.read_text().strip(),
                )

            stage = "DELIVER"
            stage_started = datetime.now(timezone.utc)
            delivered_location = self._resolve_delivery(
                marker_path, sftp, local_destination
            )
            if execution_id is not None and delivered_location is not None:
                execution_service.record_file_processing(
                    execution_id,
                    FileProcessingStep.DELIVER.value,
                    FileProcessingStatus.COMPLETED.value,
                    started_utc=stage_started,
                    file_name=marker_path.name,
                    output_file_path=delivered_location,
                )
            if execution_id is None:
                log.info(
                    f"generate_marker_for_file completed untracked "
                    f"(no execution_id): {file_path}"
                )
            return UtilityOperationResult(
                execution_id=execution_id, output_path=str(marker_path)
            )

        except Exception as exc:
            log.error(f"Marker-only orchestration failed at stage={stage}: {exc}")
            if execution_id is not None:
                execution_service.record_file_processing(
                    execution_id,
                    stage,
                    FileProcessingStatus.FAILED.value,
                    started_utc=stage_started,
                    error_message=str(exc),
                )
            raise

    @staticmethod
    def _timestamp() -> str:
        """Date-only (YYYYMMDD), matching the finalized reports' naming standard
        (docs/decisions.md 2026-09-01) - was YYYYMMDD_HHMMSS. Consequence: two
        runs of the same report on the same day produce the SAME output filename
        and overwrite each other, rather than getting distinct timestamps."""
        return datetime.now(timezone.utc).strftime("%Y%m%d")


```

tests/services/test_reporting_orchestrator.py:

```python
import dataclasses
import hashlib
import json
from datetime import datetime, timezone
from pathlib import Path

import pytest

from app.core.config import settings
from app.core.logging import log
from app.models.execution import ExecutionStatus
from app.services import execution_service
from app.services.delivery_service import (
    DeliveryConfigError,
    DeliveryError,
    DeliveryResult,
)
from app.services.download_service import DownloadResult
from app.services.fenergo_service import StatusResult, SubmitResult
from app.services.report_definition_service import (
    ReportDefinition,
    ReportDefinitionService,
)
from app.services.reporting_orchestrator import (
    OrchestrationError,
    ReportingOrchestrator,
    UtilityOperationResult,
)

CHINA_GTT_CSV = (
    "Fenergo ID,Client Name,LEI,Global Risk Rating,Scheduled Review Date,"
    "China Risk Rating,China Risk Rating (Override),China Next Review Date,"
    "China Next Review Date (Override),China Comments,Country of Incorporation,"
    "Product Category,Product Type,Booking Entity,Arranging Entity,Product ID\n"
    "1,Acme Corp,LEI123,High,2026-01-01,Medium,,2026-06-01,,,China,FamilyA,"
    "TypeA,EntityA,EntityB,PROD-1\n"
)


class FakeFenergoService:
    def __init__(
        self, status="Completed", presigned_url="https://example.test/report.csv"
    ):
        self.status = status
        self.presigned_url = presigned_url
        self.submit_calls = []

    async def submit(self, source, description):
        self.submit_calls.append((source, description))
        return SubmitResult(report_id="FEN-123")

    async def check_status(self, report_id):
        return StatusResult(status=self.status, presigned_url=self.presigned_url)


class FakeDownloadService:
    def __init__(self, csv_content=CHINA_GTT_CSV):
        self._csv_content = csv_content
        self.download_calls = []

    async def download(self, presigned_url, destination_filename):
        self.download_calls.append((presigned_url, destination_filename))
        path = settings.download_path / destination_filename
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_text(self._csv_content)
        return DownloadResult(local_path=path, size_bytes=len(self._csv_content))


class FailingDownloadService:
    async def download(self, presigned_url, destination_filename):
        raise RuntimeError("simulated download failure")


class FakeDeliveryService:
    def __init__(self, fail: bool = False):
        self.deliver_calls = []
        self._fail = fail

    def deliver(
        self, local_path, remote_directory, remote_filename=None, connection=None
    ):
        self.deliver_calls.append((local_path, remote_directory, connection))
        if self._fail:
            raise DeliveryError("simulated delivery failure")
        return DeliveryResult(
            remote_path=f"{remote_directory}/{local_path.name}",
            size_bytes=local_path.stat().st_size,
            delivered_at=datetime.now(timezone.utc),
        )


@pytest.fixture(autouse=True)
def _isolated_download_folder(tmp_path, monkeypatch):
    monkeypatch.setattr(settings, "DOWNLOAD_FOLDER", str(tmp_path / "downloads"))
    monkeypatch.setattr(settings, "SFTP_SUCCESS_DIRECTORY", "success")
    monkeypatch.setattr(settings, "AUDIT_ROOT", None)
    # ChinaGTTReport declares sftp_connection="RegCentral" - required env vars for
    # SFTPConnectionService.get() to resolve without raising.
    monkeypatch.setenv("SFTP_REGCENTRAL_HOST", "test-host")
    monkeypatch.setenv("SFTP_REGCENTRAL_USERNAME", "test-user")
    monkeypatch.setenv("SFTP_REGCENTRAL_PRIVATE_KEY_PATH", "/test/key")
    monkeypatch.setenv("SFTP_REGCENTRAL_REMOTE_FOLDER", "./RegCentralTest")


def _seed_full_run_execution(report_name="ChinaGTTReport", downloaded_file_path=None):
    """Seeds a real FULL_RUN execution the way submit->download would leave it -
    the shape every linked utility-flow test needs to attach to."""
    execution = execution_service.create_execution(report_name)
    execution_service.mark_submitted(execution.execution_id, "FEN-123")
    if downloaded_file_path is not None:
        execution_service.mark_downloaded(
            execution.execution_id, str(downloaded_file_path)
        )
    return execution


async def test_run_report_full_success_sequences_all_steps_and_completes(db):
    fenergo = FakeFenergoService()
    download = FakeDownloadService()
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(
        fenergo_service=fenergo, download_service=download, delivery_service=delivery
    )

    execution = await orchestrator.run_report("ChinaGTTReport")

    assert execution.status == ExecutionStatus.SFTP_COMPLETED.value
    assert execution.fenergo_report_id == "FEN-123"
    assert execution.output_file_path is not None

    output_path = Path(execution.output_file_path)
    assert output_path.exists()
    content = output_path.read_text()
    assert "Fenergo ID" in content
    assert "TypeA" in content

    marker_path = output_path.with_suffix(output_path.suffix + ".mrk")
    assert marker_path.exists()

    assert (
        marker_path.read_text().strip()
        == hashlib.sha256(output_path.read_bytes()).hexdigest()
    )

    assert len(delivery.deliver_calls) == 2
    delivered_paths = {call[0] for call in delivery.deliver_calls}
    assert delivered_paths == {output_path, marker_path}
    # ChinaGTTReport declares sftp_connection="RegCentral" - delivery goes to
    # that connection's own remote_folder, not the legacy global SFTP_SUCCESS_DIRECTORY.
    assert all(call[1] == "./RegCentralTest" for call in delivery.deliver_calls)
    assert all(
        call[2] is not None and call[2].host == "test-host"
        for call in delivery.deliver_calls
    )


async def test_run_report_uses_fircosoft_manifest_and_output_filename_template_when_configured(
    db, monkeypatch
):
    """A report whose registry entry opts into marker_type="fircosoft_manifest"
    gets a structured JSON manifest instead of a SHA256 checksum, and its output
    filename follows output_filename_template instead of the usual
    {report_name}_{date}.csv convention - the EDLReport shape, exercised here
    against ChinaGTTReport's already-real template/SQL for test isolation."""
    manifest_definition = dataclasses.replace(
        ReportDefinitionService.get("ChinaGTTReport"),
        marker_type="fircosoft_manifest",
        output_filename_template="Fircosoft_Fcore_EDL_Data_{date}.csv",
        manifest_source_appl={"providingParty": "gbm", "appAcronym": "b8fb"},
        manifest_source_file_static={
            "ingestMetadata": "Fircosoft_EDL_Metadata_V8.xml",
            "recordCount": "",
            "fileExtension": "csv",
            "dataFileURI": "",
        },
    )
    monkeypatch.setitem(
        ReportDefinitionService._registry, "ChinaGTTReport", manifest_definition
    )

    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    execution = await orchestrator.run_report("ChinaGTTReport")

    output_path = Path(execution.output_file_path)
    today = datetime.now(timezone.utc).strftime("%Y%m%d")
    assert output_path.name == f"Fircosoft_Fcore_EDL_Data_{today}.csv"

    marker_path = output_path.with_suffix(output_path.suffix + ".mrk")
    assert marker_path.exists()
    manifest = json.loads(marker_path.read_text())
    assert manifest["specVersion"] == "2.0"
    assert manifest["sourceAppl"]["appAcronym"] == "b8fb"
    assert manifest["sourceFiles"][0]["dataFileURI"] == output_path.name
    assert (
        manifest["sourceFiles"][0]["recordCount"] == "1"
    )  # CHINA_GTT_CSV has 1 data row


async def test_run_report_untracked_never_touches_execution_service(monkeypatch):
    """track=False (2026-09-16, Airflow smoke-testing): the real pipeline runs
    end-to-end - submit, poll, download, transform, marker, deliver - but not
    one execution_service call happens. Every function on the module is
    patched to raise if called at all, so any accidental DB touch fails this
    test loudly rather than silently succeeding against leftover DB state."""

    def _fail(*args, **kwargs):
        raise AssertionError("execution_service must not be called when track=False")

    for name in (
        "create_execution",
        "get_in_progress_execution",
        "mark_submitted",
        "mark_url_received",
        "mark_downloaded",
        "mark_transformed",
        "mark_completed",
        "mark_failed",
        "record_file_processing",
        "get_execution",
    ):
        monkeypatch.setattr(execution_service, name, _fail)

    fenergo = FakeFenergoService()
    download = FakeDownloadService()
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(
        fenergo_service=fenergo, download_service=download, delivery_service=delivery
    )

    execution = await orchestrator.run_report("ChinaGTTReport", track=False)

    assert execution.status == ExecutionStatus.SFTP_COMPLETED.value
    assert execution.fenergo_report_id == "FEN-123"
    assert execution.execution_id is not None
    assert Path(execution.output_file_path).exists()
    marker_path = Path(execution.output_file_path).with_suffix(".csv.mrk")
    assert marker_path.exists()
    assert len(delivery.deliver_calls) == 2


async def test_run_report_untracked_logs_completion(monkeypatch):
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    logged_messages = []
    sink_id = log.add(
        lambda message: logged_messages.append(message.record["message"]),
        level="INFO",
    )
    try:
        await orchestrator.run_report("ChinaGTTReport", track=False)
    finally:
        log.remove(sink_id)

    assert any(
        "run_report completed untracked" in msg and "ChinaGTTReport" in msg
        for msg in logged_messages
    )


async def test_run_report_untracked_failure_reraises_without_touching_db(monkeypatch):
    def _fail(*args, **kwargs):
        raise AssertionError("execution_service must not be called when track=False")

    for name in (
        "create_execution",
        "get_in_progress_execution",
        "mark_submitted",
        "mark_failed",
    ):
        monkeypatch.setattr(execution_service, name, _fail)

    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FailingDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    with pytest.raises(RuntimeError, match="simulated download failure"):
        await orchestrator.run_report("ChinaGTTReport", track=False)


async def test_run_report_output_lands_under_reports_archive_path_locally(db):
    """No AUDIT_ROOT set: settings.download_path/<archive_path>/, no Archive/Failed
    split - matches the user's own description of local behavior."""
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    execution = await orchestrator.run_report("ChinaGTTReport")

    output_path = Path(execution.output_file_path)
    assert output_path.parent == settings.download_path / "GTT" / "China"


async def test_run_report_output_lands_under_audit_root_archive_when_set(
    db, monkeypatch, tmp_path
):
    audit_root = tmp_path / "Audit"
    monkeypatch.setattr(settings, "AUDIT_ROOT", str(audit_root))
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    execution = await orchestrator.run_report("ChinaGTTReport")

    output_path = Path(execution.output_file_path)
    assert output_path.parent == audit_root / "Archive" / "GTT" / "China"


async def test_run_report_moves_output_to_failed_when_delivery_fails_and_audit_root_set(
    db, monkeypatch, tmp_path
):
    audit_root = tmp_path / "Audit"
    monkeypatch.setattr(settings, "AUDIT_ROOT", str(audit_root))
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(fail=True),
    )

    with pytest.raises(DeliveryError):
        await orchestrator.run_report("ChinaGTTReport")

    archive_dir = audit_root / "Archive" / "GTT" / "China"
    failed_dir = audit_root / "Failed" / "GTT" / "China"
    assert list(archive_dir.glob("*.csv")) == []
    failed_files = list(failed_dir.glob("*.csv"))
    assert len(failed_files) == 1
    assert "Fenergo ID" in failed_files[0].read_text()


async def test_run_report_leaves_output_in_place_when_delivery_fails_and_no_audit_root(
    db,
):
    """Locally (no AUDIT_ROOT), Archive/Failed are the same directory - moving is a
    harmless no-op, not an error."""
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(fail=True),
    )

    with pytest.raises(DeliveryError):
        await orchestrator.run_report("ChinaGTTReport")

    output_dir = settings.download_path / "GTT" / "China"
    files = list(output_dir.glob("*.csv"))
    assert len(files) == 1
    assert "Fenergo ID" in files[0].read_text()


async def test_run_report_full_success_records_seven_file_processing_rows_one_per_artifact(
    db,
):
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    execution = await orchestrator.run_report("ChinaGTTReport")

    records = execution_service.list_file_processing(execution.execution_id)
    steps = [r.processing_step for r in records]
    assert steps == [
        "SUBMIT",
        "POLL",
        "DOWNLOAD",
        "TRANSFORM",
        "MARKER",
        "DELIVER",
        "DELIVER",
    ]
    assert all(r.status == "COMPLETED" for r in records)

    marker_record = records[4]
    output_path = Path(execution.output_file_path)
    marker_path = output_path.with_suffix(output_path.suffix + ".mrk")
    assert marker_record.checksum_value == marker_path.read_text().strip()

    deliver_file_names = {
        r.file_name for r in records if r.processing_step == "DELIVER"
    }
    assert deliver_file_names == {output_path.name, marker_path.name}


async def test_run_report_skips_marker_records_five_file_processing_rows(
    db, monkeypatch
):
    definition = ReportDefinitionService.get("ChinaGTTReport")
    monkeypatch.setitem(
        ReportDefinitionService._registry,
        "ChinaGTTReport",
        dataclasses.replace(definition, generate_marker=False),
    )
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    execution = await orchestrator.run_report("ChinaGTTReport")

    records = execution_service.list_file_processing(execution.execution_id)
    assert [r.processing_step for r in records] == [
        "SUBMIT",
        "POLL",
        "DOWNLOAD",
        "TRANSFORM",
        "DELIVER",
    ]
    assert all(r.status == "COMPLETED" for r in records)


async def test_run_report_marks_failed_when_polling_reports_failed(db):
    fenergo = FakeFenergoService(status="Failed")
    orchestrator = ReportingOrchestrator(
        fenergo_service=fenergo,
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    with pytest.raises(OrchestrationError):
        await orchestrator.run_report("ChinaGTTReport")

    executions = [e for e in _all_executions() if e.report_name == "ChinaGTTReport"]
    execution = executions[-1]
    assert execution.status == ExecutionStatus.FAILED.value
    assert execution.failure_stage == "POLL"


async def test_run_report_marks_failed_when_download_raises(db):
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FailingDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    with pytest.raises(RuntimeError):
        await orchestrator.run_report("ChinaGTTReport")

    execution = _all_executions()[-1]
    assert execution.status == ExecutionStatus.FAILED.value
    assert execution.failure_stage == "DOWNLOAD"
    assert "simulated download failure" in execution.error_message

    records = execution_service.list_file_processing(execution.execution_id)
    assert [r.processing_step for r in records] == ["SUBMIT", "POLL", "DOWNLOAD"]
    assert records[-1].status == "FAILED"


async def test_run_report_marks_failed_when_marker_generation_raises(db):
    class FailingMarkerService:
        @staticmethod
        def create_marker_for_file(path):
            raise OSError("simulated marker generation failure")

    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=delivery,
        marker_service=FailingMarkerService,
    )

    with pytest.raises(OSError):
        await orchestrator.run_report("ChinaGTTReport")

    assert delivery.deliver_calls == []
    execution = _all_executions()[-1]
    assert execution.status == ExecutionStatus.FAILED.value
    assert execution.failure_stage == "MARKER"

    records = execution_service.list_file_processing(execution.execution_id)
    assert [r.processing_step for r in records] == [
        "SUBMIT",
        "POLL",
        "DOWNLOAD",
        "TRANSFORM",
        "MARKER",
    ]
    assert records[-1].status == "FAILED"
    assert "simulated marker generation failure" in records[-1].error_message


async def test_run_report_raises_delivery_config_error_when_success_directory_unset(
    db, monkeypatch
):
    # ChinaGTTReport now delivers via sftp_connection="RegCentral" (see
    # SFTPConnectionService), not the legacy global SFTP_SUCCESS_DIRECTORY - the
    # connection's own required env var is what needs to be missing to reproduce
    # this config error for this report now.
    monkeypatch.delenv("SFTP_REGCENTRAL_HOST", raising=False)
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=delivery,
    )

    with pytest.raises(DeliveryConfigError):
        await orchestrator.run_report("ChinaGTTReport")

    assert delivery.deliver_calls == []
    execution = _all_executions()[-1]
    assert execution.status == ExecutionStatus.FAILED.value
    assert execution.failure_stage == "DELIVER"

    records = execution_service.list_file_processing(execution.execution_id)
    assert [r.processing_step for r in records] == [
        "SUBMIT",
        "POLL",
        "DOWNLOAD",
        "TRANSFORM",
        "MARKER",
        "DELIVER",
    ]
    assert [r.status for r in records] == [
        "COMPLETED",
        "COMPLETED",
        "COMPLETED",
        "COMPLETED",
        "COMPLETED",
        "FAILED",
    ]


async def test_run_report_explicit_true_override_forces_marker_over_false_default(
    db, monkeypatch
):
    definition = ReportDefinitionService.get("ChinaGTTReport")
    monkeypatch.setitem(
        ReportDefinitionService._registry,
        "ChinaGTTReport",
        dataclasses.replace(definition, generate_marker=False),
    )
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=delivery,
    )

    execution = await orchestrator.run_report("ChinaGTTReport", generate_marker=True)

    output_path = Path(execution.output_file_path)
    marker_path = output_path.with_suffix(output_path.suffix + ".mrk")
    assert marker_path.exists()
    assert len(delivery.deliver_calls) == 2


async def test_run_report_explicit_false_override_suppresses_marker_over_true_default(
    db,
):
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=delivery,
    )

    execution = await orchestrator.run_report("ChinaGTTReport", generate_marker=False)

    output_path = Path(execution.output_file_path)
    marker_path = output_path.with_suffix(output_path.suffix + ".mrk")
    assert not marker_path.exists()
    assert len(delivery.deliver_calls) == 1


async def test_run_report_no_sftp_no_output_path_skips_delivery_stays_transformed(db):
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=delivery,
    )

    execution = await orchestrator.run_report("ChinaGTTReport", sftp=False)

    assert execution.status == ExecutionStatus.TRANSFORMED.value
    assert delivery.deliver_calls == []
    records = execution_service.list_file_processing(execution.execution_id)
    assert [r.processing_step for r in records] == [
        "SUBMIT",
        "POLL",
        "DOWNLOAD",
        "TRANSFORM",
        "MARKER",
    ]


async def test_run_report_output_path_copies_locally_and_completes(db, tmp_path):
    destination = tmp_path / "local_drop"
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    execution = await orchestrator.run_report(
        "ChinaGTTReport", local_destination=destination
    )

    assert execution.status == ExecutionStatus.SFTP_COMPLETED.value
    output_path = Path(execution.output_file_path)
    marker_path = output_path.with_suffix(output_path.suffix + ".mrk")
    assert (destination / output_path.name).exists()
    assert (destination / marker_path.name).exists()
    assert (destination / output_path.name).read_bytes() == output_path.read_bytes()

    records = execution_service.list_file_processing(execution.execution_id)
    deliver_records = [r for r in records if r.processing_step == "DELIVER"]
    assert len(deliver_records) == 2
    assert {r.output_file_path for r in deliver_records} == {
        str(destination / output_path.name),
        str(destination / marker_path.name),
    }


async def test_run_report_sftp_true_and_output_path_together_raises(db, tmp_path):
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    with pytest.raises(ValueError):
        await orchestrator.run_report(
            "ChinaGTTReport", sftp=True, local_destination=tmp_path / "out"
        )


async def test_submit_report_success_creates_and_marks_submitted(db):
    orchestrator = ReportingOrchestrator(fenergo_service=FakeFenergoService())

    execution = await orchestrator.submit_report("ChinaGTTReport")

    assert execution.status == ExecutionStatus.REQUEST_SUBMITTED.value
    assert execution.report_name == "ChinaGTTReport"
    assert execution.fenergo_report_id == "FEN-123"

    records = execution_service.list_file_processing(execution.execution_id)
    assert len(records) == 1
    assert records[0].processing_step == "SUBMIT"
    assert records[0].status == "COMPLETED"


async def test_submit_report_resumes_in_progress_execution_instead_of_resubmitting(db):
    fenergo = FakeFenergoService()
    orchestrator = ReportingOrchestrator(fenergo_service=fenergo)
    first = await orchestrator.submit_report("ChinaGTTReport")

    second = await orchestrator.submit_report("ChinaGTTReport")

    assert second.execution_id == first.execution_id
    assert len(fenergo.submit_calls) == 1


async def test_submit_report_submits_fresh_when_prior_execution_is_terminal(db):
    fenergo = FakeFenergoService()
    orchestrator = ReportingOrchestrator(fenergo_service=fenergo)
    first = await orchestrator.submit_report("ChinaGTTReport")
    execution_service.mark_completed(first.execution_id, "/tmp/out.csv")

    second = await orchestrator.submit_report("ChinaGTTReport")

    assert second.execution_id != first.execution_id
    assert len(fenergo.submit_calls) == 2


async def test_submit_report_marks_failed_when_fenergo_raises(db):
    class FailingFenergoService:
        async def submit(self, source, description):
            raise RuntimeError("simulated submit failure")

    orchestrator = ReportingOrchestrator(fenergo_service=FailingFenergoService())

    with pytest.raises(RuntimeError):
        await orchestrator.submit_report("ChinaGTTReport")

    execution = _all_executions()[-1]
    assert execution.status == ExecutionStatus.FAILED.value
    assert execution.failure_stage == "SUBMIT"

    records = execution_service.list_file_processing(execution.execution_id)
    assert len(records) == 1
    assert records[0].processing_step == "SUBMIT"
    assert records[0].status == "FAILED"


async def test_poll_execution_success_returns_presigned_url_and_marks_url_received(db):
    created = execution_service.create_execution("ChinaGTTReport")
    execution_service.mark_submitted(created.execution_id, "FEN-123")
    orchestrator = ReportingOrchestrator(fenergo_service=FakeFenergoService())

    poll_result = await orchestrator.poll_execution(created.execution_id)

    assert poll_result.status == "Completed"
    assert poll_result.presigned_url == "https://example.test/report.csv"
    updated = execution_service.get_execution(created.execution_id)
    assert updated.status == ExecutionStatus.REPORT_READY.value

    records = execution_service.list_file_processing(created.execution_id)
    assert len(records) == 1
    assert records[0].processing_step == "POLL"
    assert records[0].status == "COMPLETED"


async def test_poll_execution_marks_failed_when_status_is_failed(db):
    created = execution_service.create_execution("ChinaGTTReport")
    execution_service.mark_submitted(created.execution_id, "FEN-123")
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(status="Failed")
    )

    with pytest.raises(OrchestrationError):
        await orchestrator.poll_execution(created.execution_id)

    updated = execution_service.get_execution(created.execution_id)
    assert updated.status == ExecutionStatus.FAILED.value
    assert updated.failure_stage == "POLL"

    records = execution_service.list_file_processing(created.execution_id)
    assert len(records) == 1
    assert records[0].processing_step == "POLL"
    assert records[0].status == "FAILED"


async def test_poll_execution_raises_lookuperror_directly_for_unknown_execution_id(db):
    orchestrator = ReportingOrchestrator(fenergo_service=FakeFenergoService())

    with pytest.raises(LookupError):
        await orchestrator.poll_execution("does-not-exist")


async def test_poll_execution_once_true_completes_immediately_without_looping(db):
    created = execution_service.create_execution("ChinaGTTReport")
    execution_service.mark_submitted(created.execution_id, "FEN-123")
    orchestrator = ReportingOrchestrator(fenergo_service=FakeFenergoService())

    poll_result = await orchestrator.poll_execution(created.execution_id, once=True)

    assert poll_result.status == "Completed"
    assert poll_result.presigned_url == "https://example.test/report.csv"
    updated = execution_service.get_execution(created.execution_id)
    assert updated.status == ExecutionStatus.REPORT_READY.value
    assert updated.poll_attempts == 1

    records = execution_service.list_file_processing(created.execution_id)
    assert len(records) == 1
    assert records[0].processing_step == "POLL"


async def test_poll_execution_once_true_marks_failed_when_status_is_failed(db):
    created = execution_service.create_execution("ChinaGTTReport")
    execution_service.mark_submitted(created.execution_id, "FEN-123")
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(status="Failed")
    )

    with pytest.raises(OrchestrationError):
        await orchestrator.poll_execution(created.execution_id, once=True)

    updated = execution_service.get_execution(created.execution_id)
    assert updated.status == ExecutionStatus.FAILED.value
    assert updated.failure_stage == "POLL"

    records = execution_service.list_file_processing(created.execution_id)
    assert len(records) == 1
    assert records[0].status == "FAILED"


async def test_poll_execution_once_true_returns_pending_without_raising_or_marking_failed(
    db,
):
    created = execution_service.create_execution("ChinaGTTReport")
    execution_service.mark_submitted(created.execution_id, "FEN-123")
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(status="Pending")
    )

    poll_result = await orchestrator.poll_execution(created.execution_id, once=True)

    assert poll_result.status == "Pending"
    updated = execution_service.get_execution(created.execution_id)
    # increment_poll_attempt() sets status=POLLING as a side effect - correct,
    # not FAILED, which is the actual thing being asserted here.
    assert updated.status == ExecutionStatus.POLLING.value
    assert updated.failure_stage is None
    assert updated.poll_attempts == 1

    # "still pending" writes no file_processing row at all - not an event worth
    # recording, same treatment as "not a failure".
    assert execution_service.list_file_processing(created.execution_id) == []


async def test_poll_execution_once_true_increments_poll_attempts_across_calls(db):
    created = execution_service.create_execution("ChinaGTTReport")
    execution_service.mark_submitted(created.execution_id, "FEN-123")
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(status="Pending")
    )

    await orchestrator.poll_execution(created.execution_id, once=True)
    await orchestrator.poll_execution(created.execution_id, once=True)

    updated = execution_service.get_execution(created.execution_id)
    assert updated.poll_attempts == 2


async def test_poll_execution_marks_failed_when_no_fenergo_report_id(db):
    created = execution_service.create_execution("ChinaGTTReport")
    orchestrator = ReportingOrchestrator(fenergo_service=FakeFenergoService())

    with pytest.raises(ValueError):
        await orchestrator.poll_execution(created.execution_id)

    updated = execution_service.get_execution(created.execution_id)
    assert updated.status == ExecutionStatus.FAILED.value
    assert updated.failure_stage == "POLL"


def test_poll_execution_no_fenergo_report_id_does_not_record_file_processing(db):
    import asyncio

    created = execution_service.create_execution("ChinaGTTReport")
    orchestrator = ReportingOrchestrator(fenergo_service=FakeFenergoService())

    with pytest.raises(ValueError):
        asyncio.run(orchestrator.poll_execution(created.execution_id))

    assert execution_service.list_file_processing(created.execution_id) == []


async def test_download_execution_success(db):
    created = execution_service.create_execution("ChinaGTTReport")
    execution_service.mark_submitted(created.execution_id, "FEN-123")
    download = FakeDownloadService()
    orchestrator = ReportingOrchestrator(download_service=download)

    execution = await orchestrator.download_execution(
        created.execution_id, "https://example.test/report.csv"
    )

    assert execution.status == ExecutionStatus.DOWNLOADED.value
    assert execution.downloaded_file_path is not None
    assert download.download_calls == [
        ("https://example.test/report.csv", "FEN-123.csv")
    ]

    records = execution_service.list_file_processing(created.execution_id)
    assert len(records) == 1
    assert records[0].processing_step == "DOWNLOAD"
    assert records[0].status == "COMPLETED"


async def test_download_execution_marks_failed_when_download_raises(db):
    created = execution_service.create_execution("ChinaGTTReport")
    execution_service.mark_submitted(created.execution_id, "FEN-123")
    orchestrator = ReportingOrchestrator(download_service=FailingDownloadService())

    with pytest.raises(RuntimeError):
        await orchestrator.download_execution(created.execution_id, "https://x/y.csv")

    updated = execution_service.get_execution(created.execution_id)
    assert updated.status == ExecutionStatus.FAILED.value
    assert updated.failure_stage == "DOWNLOAD"

    records = execution_service.list_file_processing(created.execution_id)
    assert len(records) == 1
    assert records[0].status == "FAILED"


async def test_download_execution_no_fenergo_report_id_does_not_record_file_processing(
    db,
):
    created = execution_service.create_execution("ChinaGTTReport")
    orchestrator = ReportingOrchestrator(download_service=FakeDownloadService())

    with pytest.raises(ValueError):
        await orchestrator.download_execution(created.execution_id, "https://x/y.csv")

    assert execution_service.list_file_processing(created.execution_id) == []


async def test_transform_execution_success(db, tmp_path):
    input_path = tmp_path / "existing.csv"
    input_path.write_text(CHINA_GTT_CSV)
    created = execution_service.create_execution("ChinaGTTReport")
    execution_service.mark_downloaded(created.execution_id, str(input_path))
    orchestrator = ReportingOrchestrator()

    execution = await orchestrator.transform_execution(created.execution_id)

    assert execution.status == ExecutionStatus.TRANSFORMED.value
    output_path = Path(execution.output_file_path)
    assert output_path.exists()
    assert "Fenergo ID" in output_path.read_text()

    records = execution_service.list_file_processing(created.execution_id)
    assert len(records) == 1
    assert records[0].processing_step == "TRANSFORM"
    assert records[0].status == "COMPLETED"
    assert records[0].output_file_path == str(output_path)


async def test_transform_execution_marks_failed_when_no_downloaded_file_path(db):
    created = execution_service.create_execution("ChinaGTTReport")
    orchestrator = ReportingOrchestrator()

    with pytest.raises(ValueError):
        await orchestrator.transform_execution(created.execution_id)

    updated = execution_service.get_execution(created.execution_id)
    assert updated.status == ExecutionStatus.FAILED.value
    assert updated.failure_stage == "TRANSFORM"

    records = execution_service.list_file_processing(created.execution_id)
    assert len(records) == 1
    assert records[0].status == "FAILED"


# --- Utility flows: deliver_file (Flow: raw SFTP delivery, no report) ---


def test_deliver_file_unlinked_returns_untracked_result_and_writes_no_db_rows(
    db, tmp_path
):
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("a,b\n1,2\n")
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    result = orchestrator.deliver_file(source_file)

    assert isinstance(result, UtilityOperationResult)
    assert result.execution_id is None
    assert result.output_path == "success/arbitrary.csv"
    assert len(delivery.deliver_calls) == 1
    assert _all_executions() == []


def test_deliver_file_linked_writes_deliver_file_processing_row(db, tmp_path):
    execution = _seed_full_run_execution()
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("a,b\n1,2\n")
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    result = orchestrator.deliver_file(source_file, execution_id=execution.execution_id)

    assert result.execution_id == execution.execution_id
    assert result.output_path == "success/arbitrary.csv"
    records = execution_service.list_file_processing(execution.execution_id)
    assert len(records) == 1
    assert records[0].processing_step == "DELIVER"
    assert records[0].status == "COMPLETED"
    assert records[0].file_name == "arbitrary.csv"
    # The linked execution's own report_execution row is untouched.
    unchanged = execution_service.get_execution(execution.execution_id)
    assert unchanged.status == ExecutionStatus.REQUEST_SUBMITTED.value


def test_deliver_file_to_error_target(db, tmp_path, monkeypatch):
    monkeypatch.setattr(settings, "SFTP_ERROR_DIRECTORY", "error")
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("a,b\n1,2\n")
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    orchestrator.deliver_file(source_file, target="error")

    assert delivery.deliver_calls[0][1] == "error"


def test_deliver_file_linked_marks_failed_file_processing_on_error(
    db, tmp_path, monkeypatch
):
    execution = _seed_full_run_execution()
    monkeypatch.setattr(settings, "SFTP_ERROR_DIRECTORY", None)
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("a,b\n1,2\n")
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(DeliveryConfigError):
        orchestrator.deliver_file(
            source_file, target="error", execution_id=execution.execution_id
        )

    records = execution_service.list_file_processing(execution.execution_id)
    assert len(records) == 1
    assert records[0].processing_step == "DELIVER"
    assert records[0].status == "FAILED"


def test_deliver_file_unlinked_writes_no_db_rows_on_error(db, tmp_path, monkeypatch):
    monkeypatch.setattr(settings, "SFTP_ERROR_DIRECTORY", None)
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("a,b\n1,2\n")
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(DeliveryConfigError):
        orchestrator.deliver_file(source_file, target="error")

    assert _all_executions() == []


def test_deliver_file_raises_for_unknown_target(db, tmp_path):
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("a,b\n1,2\n")
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(ValueError):
        orchestrator.deliver_file(source_file, target="nonsense")


def test_deliver_file_raises_lookuperror_for_unknown_execution_id(db, tmp_path):
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("a,b\n1,2\n")
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(LookupError):
        orchestrator.deliver_file(source_file, execution_id="does-not-exist")


# --- Utility flows: generate_marker_for_file (Flow 3) ---


def test_generate_marker_for_file_unlinked_returns_untracked_result_and_writes_no_db_rows(
    db, tmp_path
):
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("some,content\n1,2\n")
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    result = orchestrator.generate_marker_for_file(source_file)

    assert isinstance(result, UtilityOperationResult)
    assert result.execution_id is None
    marker_path = source_file.with_suffix(source_file.suffix + ".mrk")
    assert result.output_path == str(marker_path)
    assert (
        marker_path.read_text().strip()
        == hashlib.sha256(source_file.read_bytes()).hexdigest()
    )
    assert len(delivery.deliver_calls) == 1
    assert _all_executions() == []


def test_generate_marker_for_file_linked_writes_marker_and_deliver_file_processing_rows(
    db, tmp_path
):
    execution = _seed_full_run_execution()
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("some,content\n1,2\n")
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    result = orchestrator.generate_marker_for_file(
        source_file, execution_id=execution.execution_id
    )

    assert result.execution_id == execution.execution_id
    records = execution_service.list_file_processing(execution.execution_id)
    assert [r.processing_step for r in records] == ["MARKER", "DELIVER"]
    assert all(r.status == "COMPLETED" for r in records)
    marker_path = source_file.with_suffix(source_file.suffix + ".mrk")
    assert records[0].checksum_value == marker_path.read_text().strip()


def test_generate_marker_for_file_no_sftp_no_output_path_skips_delivery(db, tmp_path):
    execution = _seed_full_run_execution()
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("some,content\n1,2\n")
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    result = orchestrator.generate_marker_for_file(
        source_file, execution_id=execution.execution_id, sftp=False
    )

    assert delivery.deliver_calls == []
    records = execution_service.list_file_processing(execution.execution_id)
    assert [r.processing_step for r in records] == ["MARKER"]
    marker_path = source_file.with_suffix(source_file.suffix + ".mrk")
    assert result.output_path == str(marker_path)


def test_generate_marker_for_file_output_path_copies_locally(db, tmp_path):
    execution = _seed_full_run_execution()
    destination = tmp_path / "local_drop"
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("some,content\n1,2\n")
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    orchestrator.generate_marker_for_file(
        source_file, execution_id=execution.execution_id, local_destination=destination
    )

    assert delivery.deliver_calls == []
    marker_path = source_file.with_suffix(source_file.suffix + ".mrk")
    assert (destination / marker_path.name).exists()
    records = execution_service.list_file_processing(execution.execution_id)
    deliver_record = next(r for r in records if r.processing_step == "DELIVER")
    assert deliver_record.output_file_path == str(destination / marker_path.name)


def test_generate_marker_for_file_sftp_true_and_output_path_together_raises(
    db, tmp_path
):
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("some,content\n1,2\n")
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(ValueError):
        orchestrator.generate_marker_for_file(
            source_file, sftp=True, local_destination=tmp_path / "out"
        )


def test_generate_marker_for_file_linked_writes_failed_file_processing_row_on_error(
    db, tmp_path, monkeypatch
):
    execution = _seed_full_run_execution()
    monkeypatch.setattr(settings, "SFTP_SUCCESS_DIRECTORY", None)
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("some,content\n1,2\n")
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(DeliveryConfigError):
        orchestrator.generate_marker_for_file(
            source_file, execution_id=execution.execution_id
        )

    records = execution_service.list_file_processing(execution.execution_id)
    assert [r.processing_step for r in records] == ["MARKER", "DELIVER"]
    assert [r.status for r in records] == ["COMPLETED", "FAILED"]


def test_generate_marker_for_file_unlinked_writes_no_db_rows_on_error(
    db, tmp_path, monkeypatch
):
    monkeypatch.setattr(settings, "SFTP_SUCCESS_DIRECTORY", None)
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("some,content\n1,2\n")
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(DeliveryConfigError):
        orchestrator.generate_marker_for_file(source_file)

    assert _all_executions() == []


def test_generate_marker_for_file_raises_lookuperror_for_unknown_execution_id(
    db, tmp_path
):
    source_file = tmp_path / "arbitrary.csv"
    source_file.write_text("some,content\n1,2\n")
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(LookupError):
        orchestrator.generate_marker_for_file(
            source_file, execution_id="does-not-exist"
        )


# --- Utility flows: transform_existing_csv (Flow 2) ---


async def test_transform_existing_csv_unlinked_returns_untracked_result_and_writes_no_db_rows(
    db, tmp_path
):
    input_path = tmp_path / "existing.csv"
    input_path.write_text(CHINA_GTT_CSV)
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    result = await orchestrator.transform_existing_csv("ChinaGTTReport", input_path)

    assert isinstance(result, UtilityOperationResult)
    assert result.execution_id is None
    output_path = Path(result.output_path)
    assert output_path.exists()
    assert "Fenergo ID" in output_path.read_text()
    marker_path = output_path.with_suffix(output_path.suffix + ".mrk")
    assert marker_path.exists()
    assert len(delivery.deliver_calls) == 2
    assert _all_executions() == []


async def test_transform_existing_csv_unlinked_respects_generate_marker_override_false(
    db, tmp_path
):
    input_path = tmp_path / "existing.csv"
    input_path.write_text(CHINA_GTT_CSV)
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    result = await orchestrator.transform_existing_csv(
        "ChinaGTTReport", input_path, generate_marker=False
    )

    output_path = Path(result.output_path)
    marker_path = output_path.with_suffix(output_path.suffix + ".mrk")
    assert not marker_path.exists()
    assert len(delivery.deliver_calls) == 1


async def test_transform_existing_csv_unlinked_marker_file_missing_input_raises(
    db, tmp_path
):
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(FileNotFoundError):
        await orchestrator.transform_existing_csv(
            "ChinaGTTReport", tmp_path / "does-not-exist.csv"
        )

    assert _all_executions() == []


async def test_transform_existing_csv_linked_writes_file_processing_rows(db, tmp_path):
    execution = _seed_full_run_execution(report_name="ChinaGTTReport")
    input_path = tmp_path / "existing.csv"
    input_path.write_text(CHINA_GTT_CSV)
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    result = await orchestrator.transform_existing_csv(
        "ChinaGTTReport", input_path, execution_id=execution.execution_id
    )

    assert result.execution_id == execution.execution_id
    records = execution_service.list_file_processing(execution.execution_id)
    assert [r.processing_step for r in records] == [
        "TRANSFORM",
        "MARKER",
        "DELIVER",
        "DELIVER",
    ]
    assert all(r.status == "COMPLETED" for r in records)
    # The linked execution's own report_execution row is untouched - re-processing
    # is a file-level event, not a change to the original run's outcome.
    unchanged = execution_service.get_execution(execution.execution_id)
    assert unchanged.status == ExecutionStatus.REQUEST_SUBMITTED.value
    assert unchanged.output_file_path is None


async def test_transform_existing_csv_no_sftp_no_output_path_skips_delivery(
    db, tmp_path
):
    execution = _seed_full_run_execution(report_name="ChinaGTTReport")
    input_path = tmp_path / "existing.csv"
    input_path.write_text(CHINA_GTT_CSV)
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    await orchestrator.transform_existing_csv(
        "ChinaGTTReport", input_path, execution_id=execution.execution_id, sftp=False
    )

    assert delivery.deliver_calls == []
    records = execution_service.list_file_processing(execution.execution_id)
    assert [r.processing_step for r in records] == ["TRANSFORM", "MARKER"]


async def test_transform_existing_csv_output_path_copies_locally(db, tmp_path):
    execution = _seed_full_run_execution(report_name="ChinaGTTReport")
    destination = tmp_path / "local_drop"
    input_path = tmp_path / "existing.csv"
    input_path.write_text(CHINA_GTT_CSV)
    delivery = FakeDeliveryService()
    orchestrator = ReportingOrchestrator(delivery_service=delivery)

    result = await orchestrator.transform_existing_csv(
        "ChinaGTTReport",
        input_path,
        execution_id=execution.execution_id,
        local_destination=destination,
    )

    assert delivery.deliver_calls == []
    output_path = Path(result.output_path)
    marker_path = output_path.with_suffix(output_path.suffix + ".mrk")
    assert (destination / output_path.name).exists()
    assert (destination / marker_path.name).exists()
    records = execution_service.list_file_processing(execution.execution_id)
    deliver_records = [r for r in records if r.processing_step == "DELIVER"]
    assert len(deliver_records) == 2


async def test_transform_existing_csv_sftp_true_and_output_path_together_raises(
    db, tmp_path
):
    input_path = tmp_path / "existing.csv"
    input_path.write_text(CHINA_GTT_CSV)
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(ValueError):
        await orchestrator.transform_existing_csv(
            "ChinaGTTReport", input_path, sftp=True, local_destination=tmp_path / "out"
        )


async def test_transform_existing_csv_linked_raises_on_report_name_mismatch(
    db, tmp_path
):
    execution = _seed_full_run_execution(report_name="CANDERReport")
    input_path = tmp_path / "existing.csv"
    input_path.write_text(CHINA_GTT_CSV)
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(ValueError):
        await orchestrator.transform_existing_csv(
            "ChinaGTTReport", input_path, execution_id=execution.execution_id
        )

    assert execution_service.list_file_processing(execution.execution_id) == []


async def test_transform_existing_csv_linked_writes_failed_file_processing_row_on_error(
    db, tmp_path
):
    execution = _seed_full_run_execution(report_name="ChinaGTTReport")
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(FileNotFoundError):
        await orchestrator.transform_existing_csv(
            "ChinaGTTReport",
            tmp_path / "does-not-exist.csv",
            execution_id=execution.execution_id,
        )

    records = execution_service.list_file_processing(execution.execution_id)
    assert len(records) == 1
    assert records[0].processing_step == "TRANSFORM"
    assert records[0].status == "FAILED"


async def test_transform_existing_csv_raises_lookuperror_for_unknown_execution_id(
    db, tmp_path
):
    input_path = tmp_path / "existing.csv"
    input_path.write_text(CHINA_GTT_CSV)
    orchestrator = ReportingOrchestrator(delivery_service=FakeDeliveryService())

    with pytest.raises(LookupError):
        await orchestrator.transform_existing_csv(
            "ChinaGTTReport", input_path, execution_id="does-not-exist"
        )


def test_timestamp_is_date_only_no_time_component():
    """2026-09-01: output/marker filenames switched to a date-only YYYYMMDD stamp,
    matching the finalized reports' naming standard - see docs/decisions.md."""
    value = ReportingOrchestrator._timestamp()
    assert len(value) == 8
    assert value.isdigit()
    assert datetime.strptime(value, "%Y%m%d")


async def test_output_filename_uses_date_only_timestamp(db):
    orchestrator = ReportingOrchestrator(
        fenergo_service=FakeFenergoService(),
        download_service=FakeDownloadService(),
        delivery_service=FakeDeliveryService(),
    )

    execution = await orchestrator.run_report("ChinaGTTReport")

    output_path = Path(execution.output_file_path)
    today = datetime.now(timezone.utc).strftime("%Y%m%d")
    assert output_path.name == f"ChinaGTTReport_{today}.csv"


def _all_executions():
    return execution_service.list_executions()

```

assets/templates/EDL_Report.xml:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- Source=Target deliberately identical for now - real Target renames come later,
     once transform is actually needed (see docs/decisions.md). -->
<ExtractConfig ExtractType="CSV" DateFormat="M/d/yyyy h:mm:ss tt" MaxRetry="5">
    <RecordTemplate>
        <Column Assignment="1" Source="unique_identification_number" Target="unique_identification_number" DataType="String" />
        <Column Assignment="2" Source="fenergo_id" Target="fenergo_id" DataType="String" />
        <Column Assignment="3" Source="legal_entity_name" Target="legal_entity_name" DataType="String" />
        <Column Assignment="4" Source="legal_entity_type" Target="legal_entity_type" DataType="String" />
        <Column Assignment="5" Source="entity_type" Target="entity_type" DataType="String" />
        <Column Assignment="6" Source="legal_entity_category" Target="legal_entity_category" DataType="String" />
        <Column Assignment="7" Source="lei" Target="lei" DataType="String" />
        <Column Assignment="8" Source="legal_entity_role" Target="legal_entity_role" DataType="String" />
        <Column Assignment="9" Source="legal_entity_role_status" Target="legal_entity_role_status" DataType="String" />
        <Column Assignment="10" Source="lerole_last_updated_date" Target="lerole_last_updated_date" DataType="String" />
        <Column Assignment="11" Source="lerole_last_updated_by" Target="lerole_last_updated_by" DataType="String" />
        <Column Assignment="12" Source="alias_1" Target="alias_1" DataType="String" />
        <Column Assignment="13" Source="alias_1_type" Target="alias_1_type" DataType="String" />
        <Column Assignment="14" Source="alias_2" Target="alias_2" DataType="String" />
        <Column Assignment="15" Source="alias_2_type" Target="alias_2_type" DataType="String" />
        <Column Assignment="16" Source="alias_3" Target="alias_3" DataType="String" />
        <Column Assignment="17" Source="alias_3_type" Target="alias_3_type" DataType="String" />
        <Column Assignment="18" Source="alias_4" Target="alias_4" DataType="String" />
        <Column Assignment="19" Source="alias_4_type" Target="alias_4_type" DataType="String" />
        <Column Assignment="20" Source="le_last_updated_date" Target="le_last_updated_date" DataType="String" />
        <Column Assignment="21" Source="jurisdiction" Target="jurisdiction" DataType="String" />
        <Column Assignment="22" Source="entity_of_onboarding" Target="entity_of_onboarding" DataType="String" />
        <Column Assignment="23" Source="primary_business_activity" Target="primary_business_activity" DataType="String" />
        <Column Assignment="24" Source="primary_business_location" Target="primary_business_location" DataType="String" />
        <Column Assignment="25" Source="principal_place_of_business" Target="principal_place_of_business" DataType="String" />
        <Column Assignment="26" Source="business_markets" Target="business_markets" DataType="String" />
        <Column Assignment="27" Source="country_of_domicile" Target="country_of_domicile" DataType="String" />
        <Column Assignment="28" Source="country_of_incorporation" Target="country_of_incorporation" DataType="String" />
        <Column Assignment="29" Source="gleif_registration_authority_entityid" Target="gleif_registration_authority_entityid" DataType="String" />
        <Column Assignment="30" Source="enterprise_sic_code" Target="enterprise_sic_code" DataType="String" />
        <Column Assignment="31" Source="swift_bic" Target="swift_bic" DataType="String" />
        <Column Assignment="32" Source="tax_identifier" Target="tax_identifier" DataType="String" />
        <Column Assignment="33" Source="issuing_country" Target="issuing_country" DataType="String" />
        <Column Assignment="34" Source="date_of_incorporation" Target="date_of_incorporation" DataType="String" />
        <Column Assignment="35" Source="is_this_entity_publicly_listed" Target="is_this_entity_publicly_listed" DataType="String" />
        <Column Assignment="36" Source="legal_status" Target="legal_status" DataType="String" />
        <Column Assignment="37" Source="length_of_relationship_with_scotiabank" Target="length_of_relationship_with_scotiabank" DataType="String" />
        <Column Assignment="38" Source="aml_watch_list" Target="aml_watch_list" DataType="String" />
        <Column Assignment="39" Source="annual_revenue_usd_millions" Target="annual_revenue_usd_millions" DataType="String" />
        <Column Assignment="40" Source="business_changes_made_in_the_last_5_years" Target="business_changes_made_in_the_last_5_years" DataType="String" />
        <Column Assignment="41" Source="number_of_employees" Target="number_of_employees" DataType="String" />
        <Column Assignment="42" Source="trading_name" Target="trading_name" DataType="String" />
        <Column Assignment="43" Source="registration_body" Target="registration_body" DataType="String" />
        <Column Assignment="44" Source="australian_company_number_issued" Target="australian_company_number_issued" DataType="String" />
        <Column Assignment="45" Source="anticipated_activity_of_account" Target="anticipated_activity_of_account" DataType="String" />
        <Column Assignment="46" Source="regulated_by" Target="regulated_by" DataType="String" />
        <Column Assignment="47" Source="does_the_entity_have_a_previous_name" Target="does_the_entity_have_a_previous_name" DataType="String" />
        <Column Assignment="48" Source="ltid" Target="ltid" DataType="String" />
        <Column Assignment="49" Source="regulatory_status" Target="regulatory_status" DataType="String" />
        <Column Assignment="50" Source="nature_of_business" Target="nature_of_business" DataType="String" />
        <Column Assignment="51" Source="primary_customer_base" Target="primary_customer_base" DataType="String" />
        <Column Assignment="52" Source="primary_revenue_generating_products_and_services" Target="primary_revenue_generating_products_and_services" DataType="String" />
        <Column Assignment="53" Source="registration_number" Target="registration_number" DataType="String" />
        <Column Assignment="54" Source="payment_countries" Target="payment_countries" DataType="String" />
        <Column Assignment="55" Source="third_party_account" Target="third_party_account" DataType="String" />
        <Column Assignment="56" Source="significant_customer_countries" Target="significant_customer_countries" DataType="String" />
        <Column Assignment="57" Source="significant_revenue_countries" Target="significant_revenue_countries" DataType="String" />
        <Column Assignment="58" Source="significant_supplier_countries" Target="significant_supplier_countries" DataType="String" />
        <Column Assignment="59" Source="website" Target="website" DataType="String" />
        <Column Assignment="60" Source="scheduled_review_date" Target="scheduled_review_date" DataType="String" />
        <Column Assignment="61" Source="compliance_review_date" Target="compliance_review_date" DataType="String" />
        <Column Assignment="62" Source="le_last_updated_by" Target="le_last_updated_by" DataType="String" />
        <Column Assignment="63" Source="specialized_edd_entity_type" Target="specialized_edd_entity_type" DataType="String" />
        <Column Assignment="64" Source="specialized_edd_required" Target="specialized_edd_required" DataType="String" />
        <Column Assignment="65" Source="global_risk_rating" Target="global_risk_rating" DataType="String" />
        <Column Assignment="66" Source="australia_risk_rating" Target="australia_risk_rating" DataType="String" />
        <Column Assignment="67" Source="canada_risk_rating" Target="canada_risk_rating" DataType="String" />
        <Column Assignment="68" Source="china_risk_rating" Target="china_risk_rating" DataType="String" />
        <Column Assignment="69" Source="correspondent_banking_risk_rating" Target="correspondent_banking_risk_rating" DataType="String" />
        <Column Assignment="70" Source="hong_kong_risk_rating" Target="hong_kong_risk_rating" DataType="String" />
        <Column Assignment="71" Source="india_risk_rating" Target="india_risk_rating" DataType="String" />
        <Column Assignment="72" Source="ireland_risk_rating" Target="ireland_risk_rating" DataType="String" />
        <Column Assignment="73" Source="united_kingdom_risk_rating" Target="united_kingdom_risk_rating" DataType="String" />
        <Column Assignment="74" Source="united_states_risk_rating" Target="united_states_risk_rating" DataType="String" />
        <Column Assignment="75" Source="republic_of_korea_risk_rating" Target="republic_of_korea_risk_rating" DataType="String" />
        <Column Assignment="76" Source="singapore_risk_rating" Target="singapore_risk_rating" DataType="String" />
        <Column Assignment="77" Source="global_model_exclusions" Target="global_model_exclusions" DataType="String" />
        <Column Assignment="78" Source="pep_controlled_entity" Target="pep_controlled_entity" DataType="String" />
        <Column Assignment="79" Source="material_negative_news" Target="material_negative_news" DataType="String" />
        <Column Assignment="80" Source="negative_news_information_narrative" Target="negative_news_information_narrative" DataType="String" />
        <Column Assignment="81" Source="title" Target="title" DataType="String" />
        <Column Assignment="82" Source="first_name" Target="first_name" DataType="String" />
        <Column Assignment="83" Source="last_name" Target="last_name" DataType="String" />
        <Column Assignment="84" Source="date_of_birth" Target="date_of_birth" DataType="String" />
        <Column Assignment="85" Source="place_of_birth" Target="place_of_birth" DataType="String" />
        <Column Assignment="86" Source="citizenship" Target="citizenship" DataType="String" />
        <Column Assignment="87" Source="nationality" Target="nationality" DataType="String" />
        <Column Assignment="88" Source="country_of_residence" Target="country_of_residence" DataType="String" />
        <Column Assignment="89" Source="gender" Target="gender" DataType="String" />
        <Column Assignment="90" Source="pan_id" Target="pan_id" DataType="String" />
        <Column Assignment="91" Source="company_name__insider_or_shareholder_of" Target="company_name__insider_or_shareholder_of" DataType="String" />
        <Column Assignment="92" Source="marital_status" Target="marital_status" DataType="String" />
        <Column Assignment="93" Source="number_of_identification_document" Target="number_of_identification_document" DataType="String" />
        <Column Assignment="94" Source="document_type" Target="document_type" DataType="String" />
        <Column Assignment="95" Source="document_category" Target="document_category" DataType="String" />
        <Column Assignment="96" Source="national_insurance_number" Target="national_insurance_number" DataType="String" />
        <Column Assignment="97" Source="directors_identification_number" Target="directors_identification_number" DataType="String" />
        <Column Assignment="98" Source="insider_status" Target="insider_status" DataType="String" />
        <Column Assignment="99" Source="occupation" Target="occupation" DataType="String" />
        <Column Assignment="100" Source="pep_sanctions" Target="pep_sanctions" DataType="String" />
        <Column Assignment="101" Source="controlling_shareholder" Target="controlling_shareholder" DataType="String" />
        <Column Assignment="102" Source="employer" Target="employer" DataType="String" />
        <Column Assignment="103" Source="copy_of_id_received" Target="copy_of_id_received" DataType="String" />
        <Column Assignment="104" Source="issuing_authority_of_id_1" Target="issuing_authority_of_id_1" DataType="String" />
        <Column Assignment="105" Source="issuing_country_of_id_1" Target="issuing_country_of_id_1" DataType="String" />
        <Column Assignment="106" Source="type_of_id_1" Target="type_of_id_1" DataType="String" />
        <Column Assignment="107" Source="unique_identification_number_1" Target="unique_identification_number_1" DataType="String" />
        <Column Assignment="108" Source="issuing_authority_of_id_2" Target="issuing_authority_of_id_2" DataType="String" />
        <Column Assignment="109" Source="issuing_country_of_id_2" Target="issuing_country_of_id_2" DataType="String" />
        <Column Assignment="110" Source="type_of_id_2" Target="type_of_id_2" DataType="String" />
        <Column Assignment="111" Source="unique_identification_number_2" Target="unique_identification_number_2" DataType="String" />
        <Column Assignment="112" Source="verification_date_of_id_1" Target="verification_date_of_id_1" DataType="String" />
        <Column Assignment="113" Source="expiration_date_of_id_1" Target="expiration_date_of_id_1" DataType="String" />
        <Column Assignment="114" Source="verification_date_of_id_2" Target="verification_date_of_id_2" DataType="String" />
        <Column Assignment="115" Source="expiration_date_of_id_2" Target="expiration_date_of_id_2" DataType="String" />
        <Column Assignment="116" Source="gdpr_additional_comment" Target="gdpr_additional_comment" DataType="String" />
        <Column Assignment="117" Source="contact_primary_phone_number" Target="contact_primary_phone_number" DataType="String" />
        <Column Assignment="118" Source="le_phone_number" Target="le_phone_number" DataType="String" />
        <Column Assignment="119" Source="le_email" Target="le_email" DataType="String" />
        <Column Assignment="120" Source="contact_email" Target="contact_email" DataType="String" />
        <Column Assignment="121" Source="third_party_control" Target="third_party_control" DataType="String" />
        <Column Assignment="122" Source="product_id" Target="product_id" DataType="String" />
        <Column Assignment="123" Source="product_status" Target="product_status" DataType="String" />
        <Column Assignment="124" Source="product_category" Target="product_category" DataType="String" />
        <Column Assignment="125" Source="product_type" Target="product_type" DataType="String" />
        <Column Assignment="126" Source="booking_entity" Target="booking_entity" DataType="String" />
        <Column Assignment="127" Source="arranging_entity" Target="arranging_entity" DataType="String" />
        <Column Assignment="128" Source="product_country" Target="product_country" DataType="String" />
        <Column Assignment="129" Source="region" Target="region" DataType="String" />
        <Column Assignment="130" Source="product_risk_category" Target="product_risk_category" DataType="String" />
        <Column Assignment="131" Source="frequency_of_trading_volume" Target="frequency_of_trading_volume" DataType="String" />
        <Column Assignment="132" Source="desk" Target="desk" DataType="String" />
        <Column Assignment="133" Source="internal_desk" Target="internal_desk" DataType="String" />
        <Column Assignment="134" Source="purpose_of_account_intended_use_of_account" Target="purpose_of_account_intended_use_of_account" DataType="String" />
        <Column Assignment="135" Source="source_of_funds" Target="source_of_funds" DataType="String" />
        <Column Assignment="136" Source="source_of_funds_details" Target="source_of_funds_details" DataType="String" />
        <Column Assignment="137" Source="address_type_flag" Target="address_type_flag" DataType="String" />
        <Column Assignment="138" Source="primary_add_line_1" Target="primary_add_line_1" DataType="String" />
        <Column Assignment="139" Source="primary_add_line_2" Target="primary_add_line_2" DataType="String" />
        <Column Assignment="140" Source="primary_add_city" Target="primary_add_city" DataType="String" />
        <Column Assignment="141" Source="primary_add_state" Target="primary_add_state" DataType="String" />
        <Column Assignment="142" Source="primary_add_country" Target="primary_add_country" DataType="String" />
        <Column Assignment="143" Source="primary_add_postal_code" Target="primary_add_postal_code" DataType="String" />
        <Column Assignment="144" Source="reg_add_line_1" Target="reg_add_line_1" DataType="String" />
        <Column Assignment="145" Source="reg_add_line_2" Target="reg_add_line_2" DataType="String" />
        <Column Assignment="146" Source="reg_add_city" Target="reg_add_city" DataType="String" />
        <Column Assignment="147" Source="reg_add_state" Target="reg_add_state" DataType="String" />
        <Column Assignment="148" Source="reg_add_country" Target="reg_add_country" DataType="String" />
        <Column Assignment="149" Source="reg_add_postal_code" Target="reg_add_postal_code" DataType="String" />
        <Column Assignment="150" Source="association_id" Target="association_id" DataType="String" />
        <Column Assignment="151" Source="relationship_association_type" Target="relationship_association_type" DataType="String" />
        <Column Assignment="152" Source="control_position" Target="control_position" DataType="String" />
        <Column Assignment="153" Source="percentage_of_voting_shares" Target="percentage_of_voting_shares" DataType="String" />
    </RecordTemplate>
</ExtractConfig>

```

assets/queries/EDL_Report.sql:

```sql

```

report_definitions/edl_report.py:

```python
REPORT_NAME = "EDLReport"
SQL_FILE = "EDL_Report.sql"
TEMPLATE_FILE = "EDL_Report.xml"
# Placeholders pending team lead/infra confirmation - see docs/roadmap.md.
ARCHIVE_PATH = "EDL"
SFTP_CONNECTION = "ClientCentralData"

MARKER_TYPE = "fircosoft_manifest"
OUTPUT_FILENAME_TEMPLATE = "Fircosoft_Fcore_EDL_Data_{date}.csv"
MANIFEST_SOURCE_APPL = {
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
MANIFEST_SOURCE_FILE_STATIC = {
    "dataRention": "",
    "compressType": "",
    "ingestMetadata": "Fircosoft_EDL_Metadata_V8.xml",
    "fileCompressed": "n",
    "toCharSet": "",
    "recordCount": "",  # position only - value computed per run
    "dataFileMD5": "",
    "fromCharSet": "",
    "invalidRecordThreshold": "0",
    "fileExtension": "csv",
    "dataFileURI": "",  # position only - value computed per run
    "charSetConv": "n",
}

```
