## New

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
