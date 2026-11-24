# Phase 4: File-by-File Technical Code Explanation

This document contains a comprehensive, line-by-line and block-by-block engineering analysis of every meaningful file in the repository.

---

## Table of Files Analyzed

1. [`streamlit_app.py`](#1-streamlit_apppy)
2. [`config/config.yaml`](#2-configconfigyaml)
3. [`app/orchestrator.py`](#3-apporchestratorpy)
4. [`app/core/models.py`](#4-appcoremodelspy)
5. [`app/core/config.py`](#5-appcoreconfigpy)
6. [`app/core/events.py`](#6-appcoreeventspy)
7. [`app/core/logging.py`](#7-appcoreloggingpy)
8. [`app/core/storage.py`](#8-appcorestoragepy)
9. [`app/agents/base.py`](#9-appagentsbasepy)
10. [`app/agents/ingestion_agent.py`](#10-appagentsingestion_agentpy)
11. [`app/agents/schema_agent.py`](#11-appagentsschema_agentpy)
12. [`app/agents/quality_agent.py`](#12-appagentsquality_agentpy)
13. [`app/agents/operational_agent.py`](#13-appagentsoperational_agentpy)
14. [`app/agents/anomaly_agent.py`](#14-appagentsanomaly_agentpy)
15. [`app/agents/risk_agent.py`](#15-appagentsrisk_agentpy)
16. [`app/agents/insight_agent.py`](#16-appagentsinsight_agentpy)
17. [`app/agents/alert_agent.py`](#17-appagentsalert_agentpy)
18. [`app/llm/insight.py`](#18-appllminsightpy)
19. [`tests/test_platform.py`](#19-teststest_platformpy)
20. [`src/` & `app/main.py` (Legacy Overview)](#20-legacy-subsystem-appmainpy--src)

---

## 1. `streamlit_app.py`

### 1.1 Purpose & Role
- **Path**: `streamlit_app.py`
- **Role**: Presentation layer entry point. Renders the interactive single-page web dashboard using Streamlit and Plotly, handles user inputs (ZIP uploads, demo selections, threshold adjustments), triggers pipeline execution, and displays multi-tab analytical summaries.
- **Why It Exists**: Decouples visualization from the core pipeline logic, giving non-technical clinical users an intuitive interface.

### 1.2 Key Dependencies & Imports
```python
import io, time
from pathlib import Path
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
import streamlit as st
from app.core.models import RiskLevel
from app.orchestrator import PipelineOrchestrator
```
- **Streamlit (`st`)**: Reactive UI rendering engine.
- **Plotly (`px`, `go`)**: Interactive JavaScript charts (bar charts, pie charts, grouped risk bars).
- **PipelineOrchestrator**: The backend orchestrator invoked when the user clicks "Run Pipeline".

### 1.3 State Management & Execution Flow
- **Session State Initialization (Lines 21-26)**:
  ```python
  if "orchestrator" not in st.session_state:
      st.session_state.orchestrator = PipelineOrchestrator()
      st.session_state.pipeline_results = None
      st.session_state.last_run = None
  ```
  *Technical Rationale*: Streamlit re-executes the entire script from line 1 on every user interaction (e.g., clicking a button or sliding a threshold). Storing `orchestrator` and `pipeline_results` in `st.session_state` preserves state across reruns, preventing redundant computation.

- **Pipeline Execution Block (Lines 128-151)**:
  - If "Upload ZIP" is selected, reads uploaded bytes, writes to `temp_upload.zip`, invokes `orchestrator.run(zip_path=temp_zip)`, and guarantees cleanup via `temp_zip.unlink()`.
  - If "Demo Dataset" is selected, auto-detects `data/demo` and invokes `orchestrator.run(existing_dir=demo_dir)`.
  - Stores compiled results in `st.session_state.pipeline_results` and triggers `st.rerun()`.

- **Visual Tab Rendering (Lines 175-400)**:
  - **Executive Summary**: 5 metrics (Files Ingested, Datasets Harmonized, Anomalies Detected, Alerts Created, Insights Generated).
  - **Tab 1 (Quality)**: Plotly horizontal bar chart of quality scores by dataset, detailed DataFrame with rounded completeness/validity/consistency, and text bullets of top data issues.
  - **Tab 2 (Anomalies)**: Plotly pie chart of severity distribution (`critical`: red, `high`: orange, `medium`: yellow) and interactive data table with confidence percentages.
  - **Tab 3 (Risk)**: Risk level distribution bar chart, grouped bar charts of risk components (Quality vs Operations vs Anomaly) for high-risk datasets, and CSV download button.
  - **Tab 4 (Alerts)**: Collapsible `st.expander` cards with severity badges, descriptions, study/site tags, and recommended actions.
  - **Tab 5 (Insights)**: Natural-language AI explanatory narrative cards.
  - **Tab 6 (Operations)**: Plotly operational health status pie chart and detailed KPI overview table.

### 1.4 Complexity, Edge Cases & Interview Questions
- **Time Complexity**: $O(R)$ for rendering where $R$ is the number of rows in the summary dictionary ($< 100$ items, renders in $< 100$ ms).
- **Edge Case Handled**: Handles missing files or invalid paths gracefully by halting with `st.error()` and `st.stop()` rather than crashing with an unhandled exception.
- **Interview Question**: *How does Streamlit handle execution state, and why did you use `st.session_state`?*
  - **Answer**: Streamlit runs top-to-bottom on every widget interaction. Without `st.session_state`, modifying a slider or changing a tab would wipe out prior pipeline results and force an expensive re-run of all 8 agents. `st.session_state` acts as an in-memory session cache bound to the user's browser session.

---

## 2. `config/config.yaml`

### 2.1 Purpose & Role
- **Path**: `config/config.yaml`
- **Role**: Declarative configuration file for paths, column alias mappings, data quality thresholds, operational freshness limits, anomaly z-score cutoffs, and risk weights.
- **Why It Exists**: Eliminates hardcoded business logic. Clinical data managers can tweak alias synonyms or adjust the missingness threshold from 15% to 10% without altering Python code.

### 2.2 Deep Breakdown of Configuration Sections
```yaml
paths:
  raw_root: "data/raw"
  staging_root: "data/staging"
  curated_root: "data/curated"
  logs_root: "data/logs"
  demo_data_root: "data/demo"

schema:
  alias_map:
    subject id: subject_id
    subjectid: subject_id
    subj_id: subject_id
    site: site_id
    site id: site_id
    sitename: site_id
    visit: visit_name
    visit name: visit_name
    visit_date: visit_date
    datetime: visit_date
    date: visit_date
    adverse event: adverse_event
    sae: serious_adverse_event
    status: record_status
    metric: metric_name
    value: metric_value
  required_fields:
    - subject_id
    - site_id
    - visit_date

quality:
  missing_rate_threshold: 0.15     # Flag if >15% cells are null
  duplicate_threshold: 0.05        # Flag if >5% rows are duplicated
  outlier_zscore_threshold: 3.0    # 3-sigma outlier bound

operations:
  freshness_days_threshold: 14     # Flag if no submission in 14 days

anomaly:
  zscore_threshold: 2.5            # 2.5-sigma row volume anomaly

risk:
  weights:
    quality: 0.5                   # 50% weight on data quality
    operations: 0.25               # 25% weight on operational health
    anomaly: 0.25                  # 25% weight on statistical anomalies

alerts:
  risk_score_threshold: 60.0       # Trigger alert if risk >= 60
  quality_score_threshold: 70.0    # Trigger alert if quality < 70%
```

---

## 3. `app/orchestrator.py`

### 3.1 Purpose & Role
- **Path**: `app/orchestrator.py`
- **Role**: Master workflow coordinator. Implements the **Facade Pattern** and **Mediator Pattern**. It initializes all infrastructure dependencies, wires together the 8 agents, provides a safe execution wrapper, tracks session UUIDs, publishes pipeline lifecycle events, and compiles structured results.
- **Why It Exists**: Shields caller applications (Streamlit, CLI scripts, tests) from having to know how to initialize or wire individual agents.

### 3.2 Key Methods & Execution Logic

```python
class PipelineOrchestrator:
    def __init__(self, config_path, data_base_path, log_path):
        self.config = ConfigLoader(config_path)
        self.storage = Storage(data_base_path)
        self.event_bus = EventBus()
        self.logger = StructuredLogger("orchestrator", log_file=...)
        self._init_agents()
```

- **`_init_agents(self)` (Lines 51-95)**:
  - Injects dependencies (`storage`, `config`, `event_bus`, `logger`) into each agent constructor.
  - Ensures all agents share the same singleton instances of `Storage`, `EventBus`, and `ConfigLoader`.

- **`run(self, zip_path, existing_dir) -> Dict[str, Any]` (Lines 96-190)**:
  ```python
  session_id = str(uuid.uuid4())
  context = PipelineContext(session_id=session_id)
  ```
  1. Validates inputs: ensures either `zip_path` or `existing_dir` is supplied.
  2. Dispatches `EventType.PIPELINE_STARTED`.
  3. Prepares raw data directory (unzipping safely or copying directory tree).
  4. Sequentially executes agents 1 through 8:
     ```python
     self.ingestion_agent.execute(context)
     self.schema_agent.execute(context)
     self.quality_agent.execute(context)
     self.operational_agent.execute(context)
     self.anomaly_agent.execute(context)
     self.risk_agent.execute(context)
     self.insight_agent.execute(context)
     self.alert_agent.execute(context)
     ```
  5. Records `context.completed_at = datetime.now(timezone.utc).isoformat()`.
  6. Dispatches `EventType.PIPELINE_COMPLETED`.
  7. Calls `_compile_results(context)` to build the final response payload.
  8. In case of failure: catches `Exception`, logs structured error with stack trace, publishes `EventType.PIPELINE_FAILED`, and returns `{"success": False, "error": str(e)}`.

### 3.3 Complexity & Design Patterns
- **Design Patterns**:
  - **Facade Pattern**: Single entry point `run()` simplifies execution of a complex multi-agent system.
  - **Blackboard Pattern**: The `PipelineContext` acts as the shared central blackboard that agents inspect and enrich.
- **Complexity**: $O(\sum \text{Agent Complexities})$. Memory complexity is bounded by the size of the ingested DataFrames held in `PipelineContext.harmonized_data`.

---

## 4. `app/core/models.py`

### 4.1 Purpose & Role
- **Path**: `app/core/models.py`
- **Role**: Domain entity and value object definitions using Python dataclasses and Enums.
- **Why It Exists**: Provides strict type definitions across the pipeline, eliminating untyped dictionary bugs and ensuring clean IDE autocompletion.

### 4.2 Key Dataclasses & Enums Explained

```python
class Severity(str, Enum):
    LOW = "low"; MEDIUM = "medium"; HIGH = "high"; CRITICAL = "critical"

class RiskLevel(str, Enum):
    LOW = "low"; MODERATE = "moderate"; HIGH = "high"; CRITICAL = "critical"
```

1. **`FileMetadata`**:
   - Stores raw file telemetry: `filename`, `file_path`, `study_id`, `site_id`, `file_type`, `row_count`, `column_count`, `modification_time`, `file_size`, `checksum` (SHA256), `sheet_names`.
2. **`DataQualityMetrics`**:
   - Stores normalized quality measurements: `completeness_score`, `validity_score`, `consistency_score`, `overall_quality_score` (all floats $0.0 - 100.0$), `duplicate_count`, `outlier_count`, and string list of `issues`.
3. **`Anomaly`**:
   - Represents a detected statistical irregularity: `anomaly_id`, `anomaly_type`, `severity` (Severity enum), `metric_name`, `observed_value`, `expected_value`, `confidence` ($0.0 - 1.0$), `reason`, `study_id`, `site_id`.
4. **`RiskScore`**:
   - Stores multi-factor risk: `dataset_name`, `overall_score`, `risk_level` (RiskLevel enum), `quality_component`, `operational_component`, `anomaly_component`, `evidence` list, and `recommended_action`.
5. **`Alert`**:
   - Represents an actionable escalation ticket: `alert_id`, `severity`, `title`, `description`, `study_id`, `site_id`, `evidence`, `recommended_action`, `status` ("open"), `created_at` (ISO timestamp), `related_anomalies`.
6. **`OperationalKPI`**:
   - Stores operational performance indicators: `kpi_id`, `name`, `metric_name`, `value`, `target_value`, `status` ("normal", "warning", "critical"), `description`.
7. **`AgentResult`**:
   - Standard execution return type for any `BaseAgent`: `agent_name`, `success` (bool), `data` (dict), `warnings` (list), `errors` (list), `execution_time_seconds`.
8. **`PipelineContext`**:
   - The master state accumulator passed through the pipeline. Holds session ID, timestamps, lists of metadata, dictionaries of DataFrames, quality scores, KPIs, anomalies, risk scores, alerts, and insights.

---

## 5. `app/core/config.py`

### 5.1 Purpose & Role
- **Path**: `app/core/config.py`
- **Role**: Configuration manager that merges default configuration dictionary with disk YAML files and exposes dot-notation queries.
- **Why It Exists**: Prevents missing file crashes by embedding robust in-memory fallback defaults (`DEFAULT_CONFIG`) while permitting YAML overrides.

### 5.2 Key Implementation Details
```python
class ConfigLoader:
    def __init__(self, config_path: Optional[Path | str] = None):
        self.config_path = Path(config_path) if config_path else Path("config/config.yaml")
        self._config = dict(DEFAULT_CONFIG)
        self.load()
```
- **`_deep_update(base, update)`**:
  - Recursively updates nested dictionaries so that overriding one leaf key in YAML does not accidentally erase entire sibling branches.
- **`get(key: str, default: Any = None) -> Any`**:
  - Splits dot strings (`"schema.alias_map"`, `"risk.weights.quality"`) and navigates nested dictionaries:
  ```python
  parts = key.split(".")
  current = self._config
  for part in parts:
      if isinstance(current, dict) and part in current:
          current = current[part]
      else:
          return default
  return current
  ```
  *Complexity*: $O(D)$ where $D$ is the key depth ($D \le 4$).

---

## 6. `app/core/events.py`

### 6.1 Purpose & Role
- **Path**: `app/core/events.py`
- **Role**: Implements the **Observer Pattern / Publish-Subscribe Pattern** via `EventBus`, `EventType`, and `Event`.
- **Why It Exists**: Decouples event generation from event handling. Agents do not need to know who is listening (e.g., test harnesses, loggers, or future webhook dispatchers).

### 6.2 Key Classes & Execution
- **`EventType(str, Enum)`**:
  - Defines 11 lifecycle event tokens: `PIPELINE_STARTED`, `PIPELINE_COMPLETED`, `PIPELINE_FAILED`, `FILE_INGESTED`, `SCHEMA_HARMONIZED`, `QUALITY_ASSESSED`, `OPERATIONS_EVALUATED`, `ANOMALY_DETECTED`, `RISK_CALCULATED`, `INSIGHT_GENERATED`, `ALERT_CREATED`.
- **`Event`**:
  - Immutable payload containing `event_type`, `session_id`, `source_agent`, `data` (dict), and `timestamp` (UTC ISO string).
- **`EventBus`**:
  - Maintains `_subscribers: Dict[EventType, List[Callable]]` and `_history: List[Event]`.
  - `publish(event: Event)`: Dispatches event to all registered handler callbacks in sequence and appends to `_history`. Handlers are isolated within `try-except` blocks to prevent third-party subscriber exceptions from breaking the pipeline.
  - `get_events(event_type: Optional[EventType]) -> List[Event]`: Queries history for verification and testing.

---

## 7. `app/core/logging.py`

### 7.1 Purpose & Role
- **Path**: `app/core/logging.py`
- **Role**: Structured JSON logger providing simultaneous human-readable console logging and machine-parsable JSONL file persistence.
- **Why It Exists**: Vital for clinical trial compliance (ICH GCP / 21 CFR Part 11). Every action must be recorded with UTC timestamps, logger names, severity levels, and structured contextual metadata.

### 7.2 Implementation Highlights
- Wraps Python's standard `logging.Logger` for stdout printing.
- If `log_file` is specified (e.g., `data/logs/orchestrator.jsonl`), writes JSON records:
  ```json
  {"timestamp": "2026-11-24T08:11:47.970123+00:00", "logger": "orchestrator", "level": "INFO", "message": "Agent started: quality_agent", "context": {"session_id": "...", "agent": "quality_agent"}}
  ```
- Methods: `debug()`, `info()`, `warning()`, `error()`.

---

## 8. `app/core/storage.py`

### 8.1 Purpose & Role
- **Path**: `app/core/storage.py`
- **Role**: File-based storage abstraction layer managing directory creation, Parquet serialization, CSV export, and JSON/JSONL logging.
- **Why It Exists**: Implements the **Data Access Object (DAO) Pattern**, abstracting direct disk paths and file formats from agent business logic.

### 8.2 Key Methods Explained
- **Initialization**: Automatically creates storage directories on instantiation:
  - `data/raw/`: Untouched incoming spreadsheets.
  - `data/staging/`: Intermediate extraction files.
  - `data/curated/`: Cleaned, schema-harmonized Apache Parquet datasets organized by study subfolder.
  - `data/logs/`: Structured JSON metadata and JSONL alert logs.
- **`save_dataframe_parquet(df, name, study)`**:
  - Writes Parquet using PyArrow columnar format: `data/curated/<study>/<name>.parquet`.
- **`load_dataframe_parquet(name, study)`**:
  - Reads Parquet file back into a Pandas DataFrame if it exists; returns `None` safely if missing.
- **`append_jsonl(data, name, subdir)`**:
  - Atomically appends dictionary or list of dictionaries to `data/logs/<subdir>/<name>.jsonl`. Essential for streaming alert logs without rewriting full files.

---

## 9. `app/agents/base.py`

### 9.1 Purpose & Role
- **Path**: `app/agents/base.py`
- **Role**: Abstract base class (`BaseAgent`) defining the template lifecycle and contracts for all 8 concrete agents.
- **Why It Exists**: Implements the **Template Method Pattern**. Eliminates boilerplate code across agents by centralizing logger initialization, event bus publication, and execution start/completion logging.

### 9.2 Class Contract
```python
class BaseAgent(ABC):
    def __init__(self, name: str, event_bus: Optional[EventBus] = None, logger: Optional[StructuredLogger] = None):
        self.name = name
        self.event_bus = event_bus or EventBus()
        self.logger = logger or StructuredLogger(name)

    @abstractmethod
    def execute(self, context: PipelineContext) -> AgentResult:
        pass
```
- Provides concrete helper methods:
  - `log_start(context)`: Emits structured log at start of agent run.
  - `log_completion(context, result)`: Emits structured log on finish with success/failure status and counts.
  - `log_error(context, error_msg)`: Logs error with stack context.
  - `publish_event(event_type, session_id, data)`: Emits typed `Event` onto `EventBus`.

---

## 10. `app/agents/ingestion_agent.py`

### 10.1 Purpose & Role
- **Path**: `app/agents/ingestion_agent.py`
- **Role**: First pipeline agent. Scans raw storage directories or extracts ZIP files, parses Excel workbooks with `openpyxl`, computes cryptographic SHA256 checksums, extracts study/site tokens, and populates `context.ingestion_metadata`.
- **Why It Exists**: Establishes data provenance, guarantees file integrity, and prevents path traversal security vulnerabilities.

### 10.2 Critical Code Blocks

1. **Path Traversal Protection (`extract_zip`)**:
   ```python
   @staticmethod
   def extract_zip(zip_path: Path, extract_to: Path) -> bool:
       with zipfile.ZipFile(zip_path, "r") as zip_ref:
           for member in zip_ref.namelist():
               member_path = (extract_to / member).resolve()
               if not str(member_path).startswith(str(extract_to.resolve())):
                   raise ValueError(f"Path traversal attempt detected: {member}")
           zip_ref.extractall(extract_to)
   ```
   *Security Mechanism*: Prevents the **Zip Slip vulnerability**, where malicious archives contain filenames like `../../../../Windows/System32/malicious.dll` to escape the extraction directory.

2. **Regex Study ID Extraction (`_extract_study_id`)**:
   ```python
   match = re.search(r"\b(Study\s*\d+)\b", str(file_path), re.IGNORECASE)
   if match:
       return match.group(1).title()
   ```
   *Rationale*: Robustly matches `"Study 1"`, `"Study 11"`, `"Study1"` from either directory names or filenames.

3. **Cryptographic Checksum (`_calculate_checksum`)**:
   - Streams file bytes in 4KB chunks (`iter(lambda: f.read(4096), b"")`) through `hashlib.sha256()`. Memory footprint is constant $O(1)$ regardless of file size.

4. **Workbook Telemetry (`_process_excel_file`)**:
   - Uses `openpyxl.load_workbook(file_path, data_only=False)` to read sheet names and dimensions (`max_row - 1` for rows excluding header, `max_column`), then explicitly closes the workbook to free memory handles.

---

## 11. `app/agents/schema_agent.py`

### 11.1 Purpose & Role
- **Path**: `app/agents/schema_agent.py`
- **Role**: Second pipeline agent. Resolves clinical schema heterogeneity by translating study-specific column names into canonical snake_case identifiers, saving normalized DataFrames to Parquet, and storing them in `context.harmonized_data`.
- **Why It Exists**: Downstream agents (Quality, Operations, Risk) cannot function if each study uses different column names for the same concept.

### 11.2 Normalization Algorithm (`_normalize_column_name`)
1. Strips leading/trailing whitespace and converts to lowercase.
2. Checks for an exact match in `self.alias_map` (e.g., `"subject id"` $\to$ `"subject_id"`).
3. If not found, strips internal spaces to match variations like `"subjectid"`.
4. Fallback: replaces whitespace and hyphens with underscores (`col_lower.replace(" ", "_").replace("-", "_")`).
5. Stores clean table to Parquet via `Storage.save_dataframe_parquet()`.

---

## 12. `app/agents/quality_agent.py`

### 12.1 Purpose & Role
- **Path**: `app/agents/quality_agent.py`
- **Role**: Third pipeline agent. Evaluates data quality across 3 dimensions: **Completeness**, **Validity**, and **Consistency**. Populates `context.quality_metrics`.
- **Why It Exists**: Data quality is the foundation of clinical trial integrity. Regulatory inspections flag trials with high missingness or undetected duplicate subjects.

### 12.2 Mathematical Metrics Explained
1. **Completeness Score ($C$)**:
   $$\text{Completeness} = \left(\frac{\text{Non-Null Cells}}{\text{Total Cells}}\right) \times 100$$
   - Uses vectorized `df.notna().sum().sum()`.
2. **Validity Score ($V$)**:
   - Detects exact row duplicates via `df.duplicated().sum()`.
   $$\text{Validity} = \max\left(0, 100 - \left(\frac{\text{Duplicates}}{\text{Total Rows}} \times 100\right)\right)$$
3. **Consistency Score ($K$)**:
   - Detects numeric outliers where $|z| > 3.0$ across numeric columns.
   $$\text{Consistency} = \max\left(0, 100 - \left(\frac{\text{Outliers}}{\text{Total Rows}} \times 100\right)\right)$$
4. **Composite Quality Score**:
   $$\text{Quality Score} = (0.50 \times C) + (0.30 \times V) + (0.20 \times K)$$
   - Clamped between $0.0$ and $100.0$.

---

## 13. `app/agents/operational_agent.py`

### 13.1 Purpose & Role
- **Path**: `app/agents/operational_agent.py`
- **Role**: Fourth pipeline agent. Evaluates trial execution health, tracking data recency (freshness), record throughput, and schema harmonization rates. Populates `context.operational_kpis`.
- **Why It Exists**: A trial can have clean data but terrible operations (e.g., a hospital hasn't uploaded records in 60 days). Central monitors need early warnings of operational stagnation.

### 13.2 Generated KPIs
1. **Total Ingested Records**: Aggregates row counts across all files. Status: "normal" if $> 0$, else "warning".
2. **Data Submission Freshness**: Compares file `modification_time` to current time. If age exceeds $2 \times \text{threshold}$ (28 days), marks file as stale. Status: "critical" if $> 10$ stale files, "warning" if $> 0$, else "normal".
3. **Schema Harmonization Rate**: Ratio of successfully normalized datasets to total files. Status: "normal" if $\ge 80\%$, else "warning".
4. **Study Activity**: Per-study row volume tracker.

---

## 14. `app/agents/anomaly_agent.py`

### 14.1 Purpose & Role
- **Path**: `app/agents/anomaly_agent.py`
- **Role**: Fifth pipeline agent. Employs statistical anomaly detection to identify abnormal submission volumes, sudden spikes in duplicate records, or catastrophic data missingness. Populates `context.anomalies`.
- **Why It Exists**: Flagging sudden shifts in enrollment or data volume that could indicate clinical fraud, bulk data-entry duplication, or site dropout.

### 14.2 Small-Sample Z-Score Boundary Handling
- In standard statistics, z-score is $z = \frac{x - \mu}{\sigma}$.
- **Mathematical Limitation**: For a small sample of size $N$ (e.g., $N=3$ files), the maximum possible theoretical z-score cannot exceed $\sqrt{N-1} = \sqrt{2} \approx 1.414$. A strict threshold of $z \ge 2.5$ would mathematically *never* trigger on small studies!
- **Our Solution in Code**:
  ```python
  is_anomaly = (
      z >= self.zscore_threshold
      or (len(arr) < 10 and z >= 1.3 and (arr[i] > mean * 1.5 or arr[i] < mean * 0.5))
  )
  ```
  - For small cohorts ($N < 10$), if $z \ge 1.3$ and the value deviates by more than $50\%$ from the mean, it is correctly flagged as a volume anomaly with severity assigned based on deviation magnitude.

---

## 15. `app/agents/risk_agent.py`

### 15.1 Purpose & Role
- **Path**: `app/agents/risk_agent.py`
- **Role**: Sixth pipeline agent. Integrates signals from Quality, Operations, and Anomaly agents into unified composite risk scores ($0.0 - 100.0$) for individual studies and the overall trial. Maps scores into risk tiers and generates recommended actions. Populates `context.risk_scores`.
- **Why It Exists**: Central monitors cannot review 50 disconnected metrics. They need a single prioritized risk ranking to determine which hospital site to audit first.

### 15.2 Mathematical Formulation
$$\text{Overall Risk} = \left((100 - \bar{Q}) \times w_q\right) + (S_{\text{ops}} \times w_o) + (S_{\text{anom}} \times w_a)$$
- Where:
  - $\bar{Q}$: Average Data Quality Score across datasets ($100 - \bar{Q}$ is the quality risk penalty).
  - $S_{\text{ops}}$: Operational Risk Penalty ($25$ points per critical KPI, $10$ per warning KPI, clamped to $100$).
  - $S_{\text{anom}}$: Anomaly Risk Penalty ($30$ points per critical anomaly, $15$ per high, $5$ per medium, clamped to $100$).
  - Default weights: $w_q = 0.50$, $w_o = 0.25$, $w_a = 0.25$.
- **Risk Level Mapping**:
  - $[0, 25) \to$ **Low**
  - $[25, 50) \to$ **Moderate**
  - $[50, 75) \to$ **High**
  - $[75, 100] \to$ **Critical**

---

## 16. `app/agents/insight_agent.py`

### 16.1 Purpose & Role
- **Path**: `app/agents/insight_agent.py`
- **Role**: Seventh pipeline agent. Formulates plain-language clinical insights from detected anomalies and high-risk scores. Delegates prompt/template execution to `InsightGenerator`. Populates `context.insights`.
- **Why It Exists**: Bridges the gap between raw statistical numbers and clinical actions. Tells investigators *why* a metric failed in plain English.

---

## 17. `app/agents/alert_agent.py`

### 17.1 Purpose & Role
- **Path**: `app/agents/alert_agent.py`
- **Role**: Eighth pipeline agent. Evaluates thresholds on risk scores, quality metrics, and critical anomalies to generate structured `Alert` objects. Emits `EventType.ALERT_CREATED` and appends records to `data/logs/alerts/alerts.jsonl`. Populates `context.alerts`.
- **Why It Exists**: Converts passive analytics into active notifications for study coordinators and monitors.

---

## 18. `app/llm/insight.py`

### 18.1 Purpose & Role
- **Path**: `app/llm/insight.py`
- **Role**: Strategy Pattern interface for narrative generation.
- **Classes**:
  - `InsightGenerator` (Abstract Base Class).
  - `LocalInsightGenerator`: Fast, offline, deterministic template generator that builds clinical sentences based on variance percentages and metric names.
  - `OpenAIInsightGenerator`: Dual-compatible OpenAI client supporting modern `client.chat.completions.create` (OpenAI v1+) and legacy `openai.ChatCompletion.create` (v0), with automatic fallback to `LocalInsightGenerator` on network or quota failure.

---

## 19. `tests/test_platform.py`

### 19.1 Purpose & Role
- **Path**: `tests/test_platform.py`
- **Role**: Automated test suite containing 20 unit and end-to-end integration tests covering data models, storage, event bus, every individual agent, and full pipeline execution on synthetic trial data.
- **Why It Exists**: Guarantees zero regressions, verifies system contracts, and enables confident CI/CD deployment.

### 19.2 Test Suite Architecture
- **`TestModels`**: Tests instantiations of `FileMetadata`, `DataQualityMetrics`, `RiskScore`, `Alert`.
- **`TestStorage`**: Tests directory initialization, Parquet save/load roundtrips, JSON save/load, and JSONL append integrity.
- **`TestEventBus`**: Tests pub/sub handler callbacks and event history filtering.
- **`TestQualityAgent`**: Tests completeness calculation, duplicate row identification, and z-score outlier detection.
- **`TestAnomalyAgent`**: Tests row-count outlier detection with small-sample bounds.
- **`TestRiskAgent`**: Tests score-to-level mapping and weighted composite risk calculation.
- **`TestInsightGenerator`**: Tests local template variance strings.
- **`TestSchemaAgent`**: Tests string normalization and alias lookup.
- **`TestEndToEnd`**: Constructs synthetic clinical spreadsheets on-the-fly inside a `tempfile.TemporaryDirectory()`, runs `PipelineOrchestrator`, and asserts pipeline success and non-zero counts.

---

## 20. Legacy Subsystem (`app/main.py` & `src/`)

### 20.1 Purpose & Context
- **Path**: `app/main.py` and `src/` (`pipeline.py`, `data_ingestion.py`, `feature_engineering.py`, `ml_model.py`, `insights.py`, `config.py`).
- **Role in Project History**: Represents the earlier exploratory Phase 1 architecture that used PySpark to aggregate site features and train a logistic regression risk model, surfaced via a Flask dashboard with Jinja templates.
- **Interview Positioning**: In an interview, explain: *"The project originated with a PySpark/Flask proof-of-concept in `src/`. To eliminate JVM overhead, enable zero-infrastructure deployments, and achieve ICH E6(R2) regulatory transparency, we refactored the core into the production 8-Agent Modular Monolith in `app/agents/` with an interactive Streamlit UI."*
