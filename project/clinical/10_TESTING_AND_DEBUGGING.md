# Phase 11: Testing Strategy, Test Suite Analysis & Debugging Playbook

---

## 1. Existing Test Suite Analysis (`tests/test_platform.py`)

The platform contains an automated test suite executed via `pytest`. All 20 tests pass in **$< 1.5$ seconds**.

```
Test Suite Execution Breakdown:
============================= test session starts =============================
platform win32 -- Python 3.14.6, pytest-9.2.0, pluggy-1.7.0
rootdir: C:\...\clinical-agentic-ai-platform
configfile: pytest.ini

tests/test_platform.py::TestModels::test_file_metadata PASSED            [  5%]
tests/test_platform.py::TestModels::test_data_quality_metrics PASSED     [ 10%]
tests/test_platform.py::TestModels::test_risk_score PASSED               [ 15%]
tests/test_platform.py::TestModels::test_alert PASSED                    [ 20%]
tests/test_platform.py::TestStorage::test_storage_initialization PASSED  [ 25%]
tests/test_platform.py::TestStorage::test_save_and_load_dataframe PASSED [ 30%]
tests/test_platform.py::TestStorage::test_save_and_load_json PASSED      [ 35%]
tests/test_platform.py::TestStorage::test_append_jsonl PASSED            [ 40%]
tests/test_platform.py::TestEventBus::test_publish_and_subscribe PASSED  [ 45%]
tests/test_platform.py::TestEventBus::test_event_filtering PASSED        [ 50%]
tests/test_platform.py::TestQualityAgent::test_calculate_completeness PASSED [ 55%]
tests/test_platform.py::TestQualityAgent::test_duplicate_detection PASSED [ 60%]
tests/test_platform.py::TestQualityAgent::test_outlier_detection PASSED  [ 65%]
tests/test_platform.py::TestAnomalyAgent::test_row_count_anomaly_detection PASSED [ 70%]
tests/test_platform.py::TestRiskAgent::test_score_to_level PASSED        [ 75%]
tests/test_platform.py::TestRiskAgent::test_composite_risk_calculation PASSED [ 80%]
tests/test_platform.py::TestInsightGenerator::test_local_insight_row_count PASSED [ 85%]
tests/test_platform.py::TestInsightGenerator::test_local_insight_missingness PASSED [ 90%]
tests/test_platform.py::TestSchemaAgent::test_column_normalization PASSED [ 95%]
tests/test_platform.py::TestEndToEnd::test_pipeline_with_synthetic_data PASSED [100%]
======================== 20 passed in 1.36s ========================
```

---

## 2. Deep Dive on Key Test Implementations

### 2.1 Storage Layer Isolation (`TestStorage`)
Uses Python's `tempfile.TemporaryDirectory()` context manager:
```python
with tempfile.TemporaryDirectory() as tmpdir:
    storage = Storage(Path(tmpdir))
    df = pd.DataFrame({"col1": [1, 2, 3], "col2": ["a", "b", "c"]})
    storage.save_dataframe_parquet(df, "test_data")
    loaded = storage.load_dataframe_parquet("test_data")
    pd.testing.assert_frame_equal(df, loaded)
```
- **Why Important**: Tests never pollute the production `data/` directory. When the `with` block exits, the OS deletes all temporary files, ensuring 100% test idempotency.

### 2.2 End-to-End Integration Test (`TestEndToEnd`)
Generates synthetic clinical spreadsheets on-the-fly:
```python
df = pd.DataFrame({
    "Subject ID": ["S001", "S002", "S003"],
    "Site ID": ["Site1", "Site1", "Site2"],
    "Visit Date": ["2025-01-01", "2025-01-02", "2025-01-03"],
    "Value": [10.5, 11.2, 9.8],
})
excel_file = data_dir / "Study1" / "data.xlsx"
df.to_excel(excel_file, index=False)

orchestrator = PipelineOrchestrator(config_path=config_path, data_base_path=tmpdir / "output")
results = orchestrator.run(existing_dir=data_dir)

assert results["success"] is True
assert results["summary"]["files_ingested"] > 0
assert results["summary"]["datasets_harmonized"] > 0
```
- **Why Important**: Exercises the entire call chain: Ingestion $\to$ Schema $\to$ Quality $\to$ Operations $\to$ Anomaly $\to$ Risk $\to$ Insight $\to$ Alert without relying on external test files.

---

## 3. What Is NOT Tested (Coverage Gaps & Test Plan)

| Component / Scenario | Current Coverage Status | Risk Severity | Suggested Test Case |
| :--- | :---: | :---: | :--- |
| **Zip Slip Attack** | Not tested in automated suite | High | Create a synthetic ZIP with `../../evil.txt` and assert `ValueError` is raised. |
| **Corrupted Excel XML** | Handled in code, unverified in test | Medium | Feed a 0-byte `.xlsx` or plain text file with an `.xlsx` extension and assert graceful warning. |
| **OpenAI API Fallback** | Unverified in automated test | Medium | Mock `openai.OpenAI` to throw `APIConnectionError` and assert `LocalInsightGenerator` fallback triggers cleanly. |
| **Concurrent Pipeline Runs** | Untested | Medium | Run two `PipelineOrchestrator.run()` calls in parallel threads and assert isolated session IDs. |

---

## 4. Production Incident Debugging Playbook

### Incident 1: "DLL load failed while importing timestamps: An Application Control policy has blocked this file"
- **Root Cause**: Windows Defender Application Control (WDAC) or corporate AppLocker policies block loading unsigned compiled C extensions (`.pyd` DLLs) located inside user writable paths like `AppData\Local` or `.venv\Scripts`.
- **How to Debug**:
  1. Check Python installation path: `where.exe python`.
  2. Inspect where site-packages are loaded: `python -c "import pandas; print(pandas.__file__)"`.
  3. If blocked in `AppData\Local`, install packages to the user roaming directory using `python -m pip install --user <package>`, which is whitelisted under standard Windows corporate policies.

### Incident 2: Anomaly Agent Fails to Flag Obvious Outlier in Small Trial
- **Root Cause**: The cohort only has 3 or 4 files. As proven in Phase 8, the maximum possible z-score for $N=3$ is $\sqrt{2} \approx 1.414$, which never reaches a static $z \ge 2.5$ threshold.
- **How to Debug**:
  1. Verify sample size: `len(arr)`.
  2. Check `app/agents/anomaly_agent.py` line 108.
  3. Ensure the dual-criterion logic is active: for $N < 10$, evaluate $z \ge 1.3$ and value variance $> 1.5\mu$.

### Incident 3: Pipeline Crashes with `KeyError: 'overall_quality_score'`
- **Root Cause**: An uploaded Excel sheet was completely empty or contained no parseable data. `SchemaAgent` skipped it, so `QualityAgent` never generated metrics for that key.
- **How to Debug**:
  1. Open `data/logs/orchestrator.jsonl` and locate the failed `session_id`.
  2. Trace `context.harmonized_data.keys()` vs `context.quality_metrics.keys()`.
  3. Verify that `RiskAgent._calculate_composite_score()` provides fallback defaults when `context.quality_metrics` is empty:
     ```python
     quality_score = sum(scores) / len(scores) if scores else 50.0
     ```

### Incident 4: OpenAI Rate Limit / Quota Exceeded (`HTTP 429`)
- **Root Cause**: External OpenAI API quota expired or rate limit was hit.
- **How to Debug**:
  1. Inspect `app/llm/insight.py` line 125.
  2. Verify that the `try-except Exception as e:` block successfully intercepts the error:
     ```python
     except Exception as e:
         self.logger.warning(f"OpenAI API error: {e}. Using local generator.")
         return LocalInsightGenerator().generate_insight(...)
     ```
  3. The pipeline continues uninterrupted, emitting deterministic template insights without failing the user session.
