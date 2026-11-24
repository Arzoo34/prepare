# Phase 15: Rebuild the Project from Scratch (8-Stage Roadmap)

This roadmap outlines an 8-stage step-by-step engineering plan to rebuild this complete platform from zero without copy-pasting code.

---

## Stage 1: Clinical Domain & Spreadsheet Anatomy

### 1. What You Need to Learn
- Understand Electronic Data Capture (EDC) systems, Clinical Data Management (CDM), and ICH E6(R2) Good Clinical Practice guidelines.
- Learn why clinical spreadsheets differ across studies: CPID subject metrics, EDRR query backlogs, Serious Adverse Events (SAE), MedDRA adverse event coding, WHODD drug coding, and visit projection tracking.
- Study the OpenXML format of `.xlsx` files and how `openpyxl` parses sheets into memory.

### 2. What to Implement
- A scratch exploratory script to inspect sheet names, column headers, and row counts from sample study Excel files.

### 3. Key Concepts to Master
- Tabular heterogeneity, schema drift, subject-to-site relationships, and clinical trial milestone tracking.

### 4. Common Mistakes to Avoid
- *Mistake*: Assuming every study file uses standard column names like `subject_id` or that all dates are stored as ISO strings.

---

## Stage 2: Core Infrastructure Layer

### 1. Files to Create
- `app/core/models.py`
- `app/core/config.py`
- `app/core/events.py`
- `app/core/logging.py`
- `app/core/storage.py`

### 2. What You Need to Implement
1. **`models.py`**: Define strongly-typed dataclasses: `FileMetadata`, `DataQualityMetrics`, `Anomaly`, `RiskScore`, `Alert`, `OperationalKPI`, `AgentResult`, `PipelineContext`, and Enums: `Severity`, `RiskLevel`.
2. **`config.py`**: Create `ConfigLoader` with embedded default fallback dictionary, YAML file loading with `yaml.safe_load()`, recursive dictionary merging, and dot-notation lookup (`get("schema.alias_map")`).
3. **`events.py`**: Implement the Observer Pattern: `EventType` enum, immutable `Event` dataclass, and `EventBus` with `subscribe()`, `publish()`, and `get_events()`.
4. **`logging.py`**: Implement `StructuredLogger` outputting formatted logs to stdout and writing JSON records to `.jsonl`.
5. **`storage.py`**: Create `Storage` managing `data/raw`, `data/staging`, `data/curated`, and `data/logs`, with methods to save/load Parquet, CSV, JSON, and JSONL.

### 3. How to Test This Stage
- Write unit tests asserting that:
  - `ConfigLoader().get("quality.missing_rate_threshold")` returns `0.15`.
  - `Storage` saves a DataFrame to Parquet and loads it back identically (`pd.testing.assert_frame_equal`).
  - `EventBus` dispatches an event to a registered callback.

---

## Stage 3: Ingestion & Schema Harmonization Agents

### 1. Files to Create
- `app/agents/base.py`
- `app/agents/ingestion_agent.py`
- `app/agents/schema_agent.py`

### 2. What You Need to Implement
1. **`base.py`**: Define abstract `BaseAgent` implementing `log_start()`, `log_completion()`, `log_error()`, `publish_event()`, and abstract `execute(context)`.
2. **`ingestion_agent.py`**:
   - `extract_zip()`: Validates that member paths start with `extract_to.resolve()` to neutralize Zip Slip vulnerabilities.
   - `_calculate_checksum()`: Streams 4KB chunks through `hashlib.sha256()`.
   - `_extract_study_id()`: Regex parser `\b(Study\s*\d+)\b`.
   - Reads workbook dimensions via `openpyxl` and appends `FileMetadata` to `context.ingestion_metadata`.
3. **`schema_agent.py`**:
   - Reads raw files into Pandas, applies `alias_map` to rename columns to canonical identifiers, and saves cleaned tables to `data/curated/<study>/*.parquet`.

### 3. What You Should Be Able to Explain
- Why Parquet is superior to CSV for analytical storage.
- How Zip Slip works and how your path check defends against it.

---

## Stage 4: Data Quality & Operational Intelligence Agents

### 1. Files to Create
- `app/agents/quality_agent.py`
- `app/agents/operational_agent.py`

### 2. What You Need to Implement
1. **`quality_agent.py`**:
   - Completeness %: Vectorized non-null cell count divided by total cells.
   - Validity %: 100 minus duplicate row percentage.
   - Consistency %: 100 minus numeric outlier percentage ($|z| > 3.0$).
   - Composite Quality Score: $(0.50 \times C) + (0.30 \times V) + (0.20 \times K)$.
2. **`operational_agent.py`**:
   - Freshness tracking: Compares file modification timestamps against a 14/28-day threshold.
   - Total record volume aggregation.
   - Schema harmonization throughput percentage.

### 3. Common Mistakes to Avoid
- *Mistake*: Iterating over DataFrame rows with `for index, row in df.iterrows():`—this is 1,000x slower than Pandas vectorization. Always use vectorized operations (`df.notna()`, `df.duplicated()`).

---

## Stage 5: Anomaly Detection & Composite Risk Engine

### 1. Files to Create
- `app/agents/anomaly_agent.py`
- `app/agents/risk_agent.py`

### 2. What You Need to Implement
1. **`anomaly_agent.py`**:
   - Computes z-scores on ingested row counts.
   - Implements the dual-criterion model: for small cohorts ($N < 10$), checks $z \ge 1.3$ and variance $> 1.5\mu$ to overcome the sample variance upper bound theorem.
   - Flags low completeness ($< 60\%$) and duplicate surges ($> 50$ rows).
2. **`risk_agent.py`**:
   - Combines Quality Deficit ($100 - \bar{Q}$), Operational penalties, and Anomaly penalties into a single score ($0.0 - 100.0$).
   - Maps scores into `RiskLevel`: Low, Moderate, High, Critical.
   - Generates actionable clinical recommendations for each risk tier.

---

## Stage 6: Explainable AI Insights & Automated Alerts

### 1. Files to Create
- `app/llm/insight.py`
- `app/agents/insight_agent.py`
- `app/agents/alert_agent.py`

### 2. What You Need to Implement
1. **`insight.py`**:
   - Implements the Strategy Pattern with `InsightGenerator`.
   - `LocalInsightGenerator`: Builds deterministic clinical explanations using template strings and calculated percentage variances.
   - `OpenAIInsightGenerator`: Dual-compatible OpenAI client (supporting modern v1+ and legacy v0) with automatic fallback to `LocalInsightGenerator` on network/quota exceptions.
2. **`insight_agent.py`**: Iterates through detected anomalies and high-risk scores, calling the generator and appending text to `context.insights`.
3. **`alert_agent.py`**: Evaluates thresholds (Risk $\ge 60$, Quality $< 70\%$, Critical anomalies), creates `Alert` objects with evidence arrays, and appends records to `alerts.jsonl`.

---

## Stage 7: Pipeline Orchestrator & Interactive UI

### 1. Files to Create
- `app/orchestrator.py`
- `streamlit_app.py`

### 2. What You Need to Implement
1. **`orchestrator.py`**:
   - Implements `PipelineOrchestrator` to initialize dependencies, instantiate all 8 agents, manage directory copying/unzipping, run agents in sequence, trap errors, and compile the final summary payload.
2. **`streamlit_app.py`**:
   - Initializes `st.session_state` to preserve results across user interactions.
   - Renders sidebar controls (data source radio, threshold sliders, Run button).
   - Builds 6 interactive tabs using Plotly charts and Pandas data tables.

---

## Stage 8: Automated Test Suite & CI/CD Hardening

### 1. Files to Create
- `pytest.ini`
- `tests/test_platform.py`

### 2. What You Need to Implement
- Configure `pytest.ini` with standard options (`-v --tb=short`).
- Write 20 tests covering:
  - Models and dataclass defaults.
  - Storage Parquet/JSON/JSONL roundtrips in temporary directories.
  - EventBus publish/subscribe dispatch.
  - Quality, Anomaly, Risk, and Schema agent methods.
  - End-to-end pipeline execution with synthetically generated spreadsheets.
- Assert that `python -m pytest` executes with 100% pass rate in $< 2$ seconds.
