# Phase 3A & 3B: Complete System Architecture & Mermaid Diagrams

---

## 1. Architectural Style & Paradigm

The platform is designed as an **Event-Driven Modular Monolith with an Agentic Pipeline Pattern**. 

```
┌────────────────────────────────────────────────────────────────────────┐
│                        ARCHITECTURAL PARADIGM                          │
├──────────────────────────┬─────────────────────────────────────────────┤
│ Modular Monolith         │ Code organized into cleanly decoupled       │
│                          │ packages (app/core, app/agents, app/llm)    │
│                          │ sharing a single execution process.         │
├──────────────────────────┼─────────────────────────────────────────────┤
│ Multi-Agent Pipeline     │ 8 discrete, autonomous functional agents    │
│                          │ inheriting from BaseAgent, each responsible │
│                          │ for a single bounded domain task.           │
├──────────────────────────┼─────────────────────────────────────────────┤
│ Event-Driven Messaging   │ State changes emit structured Event objects │
│                          │ onto an EventBus for loose coupling and     │
│                          │ auditability.                               │
├──────────────────────────┼─────────────────────────────────────────────┤
│ Blackboard State Pattern │ All agents read from and mutate a shared    │
│                          │ PipelineContext object across lifecycle.    │
└──────────────────────────┴─────────────────────────────────────────────┘
```

### Why This Architecture Was Chosen
1. **Separation of Concerns**: Clinical trial intelligence encompasses diverse operations (parsing binary Excel sheets, string token mapping, statistical math, narrative prompt formatting, and disk I/O). Isolating each into an independent Agent prevents "spaghetti code."
2. **Zero Deployment Friction**: Unlike distributed microservices requiring Docker Compose, Kubernetes, and network service discovery, this architecture runs in any Python 3.10+ environment while maintaining strict internal boundaries.
3. **Deterministic Auditability**: Regulated life-sciences software requires 100% reproducible execution paths. Linear orchestration coupled with event-bus publishing provides a clear, verifiable audit trail.

---

## 2. Layered Architecture Breakdown

```mermaid
graph TD
    subgraph UI_LAYER["1. Presentation Layer"]
        StreamlitApp["Streamlit Dashboard (streamlit_app.py)"]
        FlaskUI["Flask UI (app/main.py) - Legacy"]
    end

    subgraph ORCH_LAYER["2. Orchestration Layer"]
        Orchestrator["PipelineOrchestrator (app/orchestrator.py)"]
    end

    subgraph AGENT_LAYER["3. Multi-Agent Pipeline Layer"]
        direction TB
        BaseAgent["BaseAgent (app/agents/base.py)"]
        A1["1. IngestionAgent"]
        A2["2. SchemaAgent"]
        A3["3. QualityAgent"]
        A4["4. OperationalAgent"]
        A5["5. AnomalyAgent"]
        A6["6. RiskAgent"]
        A7["7. InsightAgent"]
        A8["8. AlertAgent"]
        
        BaseAgent -.-> A1
        BaseAgent -.-> A2
        BaseAgent -.-> A3
        BaseAgent -.-> A4
        BaseAgent -.-> A5
        BaseAgent -.-> A6
        BaseAgent -.-> A7
        BaseAgent -.-> A8
    end

    subgraph CORE_LAYER["4. Core Infrastructure Layer"]
        Context["PipelineContext (app/core/models.py)"]
        Bus["EventBus & Event (app/core/events.py)"]
        Config["ConfigLoader (app/core/config.py)"]
        Logger["StructuredLogger (app/core/logging.py)"]
        LLM["InsightGenerator & Local/OpenAI (app/llm/insight.py)"]
    end

    subgraph PERSISTENCE_LAYER["5. Persistence & Storage Layer"]
        Storage["Storage Abstraction (app/core/storage.py)"]
        RawDir[("data/raw/")]
        CuratedDir[("data/curated/*.parquet")]
        LogsDir[("data/logs/*.jsonl")]
    end

    StreamlitApp --> Orchestrator
    Orchestrator --> A1 --> A2 --> A3 --> A4 --> A5 --> A6 --> A7 --> A8
    
    A1 & A2 & A3 & A4 & A5 & A6 & A7 & A8 <--> Context
    A1 & A2 & A3 & A4 & A5 & A6 & A7 & A8 --> Bus
    A1 & A2 & A3 & A4 & A5 & A6 & A7 & A8 --> Logger
    A2 & A3 & A4 & A5 & A6 & A8 --> Config
    A7 --> LLM
    A1 & A2 & A8 --> Storage
    Storage --> RawDir
    Storage --> CuratedDir
    Storage --> LogsDir
```

### Layer Responsibilities & Boundaries

1. **Presentation Layer (`streamlit_app.py`)**:
   - Manages UI controls, file upload widgets, threshold sliders, and tab navigation.
   - Converts raw Python dictionaries and lists from `PipelineContext` into Plotly visualizations and data tables.
   - Never directly touches file disks or performs math calculations; delegates everything to `PipelineOrchestrator`.

2. **Orchestration Layer (`app/orchestrator.py`)**:
   - Initializes core infrastructure (`ConfigLoader`, `Storage`, `EventBus`, `StructuredLogger`).
   - Instantiates all 8 agents and wires their dependencies.
   - Manages execution lifecycle, handles directory copying/unzipping, traps unhandled exceptions, and compiles the final executive summary.

3. **Agent Layer (`app/agents/`)**:
   - Each agent inherits from `BaseAgent`.
   - Each agent receives the mutable `PipelineContext`, executes its specialized logic, publishes lifecycle events to the `EventBus`, and returns an `AgentResult` object.

4. **Core Infrastructure Layer (`app/core/`)**:
   - `models.py`: Strongly-typed Python dataclasses and enums providing type safety across the entire application.
   - `config.py`: Encapsulates configuration parsing with recursive dictionary updates and dot-notation lookups.
   - `events.py`: Implements synchronous in-memory Publish-Subscribe messaging.
   - `logging.py`: Dual console and append-only JSON Lines structured logger.
   - `llm/insight.py`: Strategy pattern abstracting template generation from OpenAI API calls.

5. **Storage Layer (`app/core/storage.py`)**:
   - Provides a clean abstracted interface for saving and loading Parquet, CSV, JSON, and JSONL data without exposing direct `os` or `pathlib` manipulations to high-level agents.

---

## 3. Comprehensive Mermaid Diagrams

### Diagram 1: High-Level System Architecture

```mermaid
flowchart TB
    User([Clinical User / Data Manager]) -->|Uploads ZIP or Selects Demo| UI[Streamlit Web Application]
    
    subgraph SystemBoundary["Clinical Agentic AI Platform"]
        UI -->|Invokes run| Orch[Pipeline Orchestrator]
        
        subgraph PipelineEngine["Execution Pipeline Engine"]
            Orch --> Stage1[Ingestion & Verification]
            Stage1 --> Stage2[Schema Normalization]
            Stage2 --> Stage3[Quality Assessment]
            Stage3 --> Stage4[Operational Intelligence]
            Stage4 --> Stage5[Anomaly Detection]
            Stage5 --> Stage6[Composite Risk Scoring]
            Stage6 --> Stage7[Generative Insights]
            Stage7 --> Stage8[Alerting & Task Creation]
        end
        
        subgraph BusStorage["Core Bus & Storage"]
            EventBus[(In-Memory Event Bus)]
            FileStorage[(Local File Storage: Parquet/JSONL)]
        end
        
        PipelineEngine -.->|Publishes Events| EventBus
        PipelineEngine -.->|Reads & Writes| FileStorage
    end

    Orch -->|Returns Compiled Dict| UI
    UI -->|Renders 6 Tabs| User
```

#### Diagram 1 Explanation:
- **User Node**: Represents the clinical investigator, CRA, or data manager interacting via a browser.
- **UI Node**: Streamlit application capturing user input and displaying interactive results.
- **Pipeline Orchestrator**: The central controller coordinating execution.
- **PipelineEngine Stages (1 to 8)**: The linear execution pipeline where each stage builds on the results of previous stages.
- **EventBus**: Captures every state change as an immutable event for auditability.
- **FileStorage**: Local directory structure storing raw files, curated Parquet tables, and JSONL alert logs.

---

### Diagram 2: Multi-Agent Interaction & Event-Bus Topology

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Streamlit UI
    participant Orch as PipelineOrchestrator
    participant Context as PipelineContext
    participant Bus as EventBus
    participant Agents as Pipeline Agents (1-8)
    participant Storage as Storage Layer

    User->>UI: Clicks "Run Pipeline"
    UI->>Orch: orchestrator.run(existing_dir)
    Orch->>Context: Instantiate(session_id)
    Orch->>Bus: publish(PIPELINE_STARTED)
    
    loop For Each of the 8 Agents in Sequence
        Orch->>Agents: agent.execute(context)
        Agents->>Agents: log_start(context)
        Agents->>Context: Read inputs from context
        Agents->>Storage: Read/Write files (Parquet/JSONL)
        Agents->>Context: Mutate context with results
        Agents->>Bus: publish(EventType, data)
        Agents->>Agents: log_completion(context, result)
        Agents-->>Orch: return AgentResult
    end

    Orch->>Context: set completed_at
    Orch->>Bus: publish(PIPELINE_COMPLETED)
    Orch->>Orch: _compile_results(context)
    Orch-->>UI: return results dictionary
    UI-->>User: Render Dashboard Tabs
```

#### Diagram 2 Explanation:
- **Steps 1-4**: The user triggers execution; the orchestrator assigns a unique UUID `session_id`, creates the empty blackboard `PipelineContext`, and fires the `PIPELINE_STARTED` event.
- **Steps 5-13 (The Agent Loop)**: Each agent sequentially receives the shared `context`. It logs its invocation, reads prior results, performs its transformations, writes artifacts through `Storage`, updates the `context` in place, fires an event on `EventBus`, and returns an `AgentResult` containing execution metadata.
- **Steps 14-18**: The orchestrator marks completion, compiles the final payload, and Streamlit re-renders the UI reactively.

---

### Diagram 3: Complete Data Flow & Artifact Transformation

```mermaid
flowchart TD
    RawExcel["Raw Study Excel Files (*.xlsx, *.xls)"] -->|IngestionAgent| Metadata["FileMetadata<br/>(size, row_count, col_count, checksum)"]
    
    RawExcel & Metadata -->|SchemaAgent| CleanDF["Harmonized DataFrames<br/>(Normalized snake_case columns)"]
    
    CleanDF -->|Storage.save_dataframe_parquet| ParquetFiles[("data/curated/<study>/*.parquet")]
    
    CleanDF -->|QualityAgent| QualityScores["DataQualityMetrics<br/>(Completeness %, Duplicates, Outliers)"]
    
    Metadata & CleanDF -->|OperationalAgent| OperationalKPIs["OperationalKPI<br/>(Freshness, Record Volume, Harmonization Rate)"]
    
    Metadata & QualityScores -->|AnomalyAgent| Anomalies["Anomaly Records<br/>(Row Count Z-Score, Duplicate Spikes)"]
    
    QualityScores & OperationalKPIs & Anomalies -->|RiskAgent| RiskScores["RiskScore<br/>(Overall 0-100, Study-level, RiskLevel)"]
    
    Anomalies & RiskScores -->|InsightAgent + LLM| NarrativeInsights["Human-Readable AI Insights<br/>(Explaining Anomaly Root Causes)"]
    
    RiskScores & QualityScores & Anomalies -->|AlertAgent| StructuredAlerts["Alert Objects<br/>(Critical, High, Medium + Recommendations)"]
    
    StructuredAlerts -->|Storage.append_jsonl| AlertLog[("data/logs/alerts/alerts.jsonl")]
    
    Metadata & QualityScores & OperationalKPIs & Anomalies & RiskScores & NarrativeInsights & StructuredAlerts --> StreamlitDashboard["Streamlit Executive Dashboard"]
```

#### Diagram 3 Explanation:
- Demonstrates how raw, untrusted data flows progressively through the pipeline.
- Raw Excel workbooks are transformed into structured in-memory `DataFrame` objects.
- Normalized tables are permanently persisted into compressed columnar Parquet files (`data/curated/`).
- Statistical metrics (`DataQualityMetrics`, `OperationalKPI`, `Anomaly`) feed into the composite `RiskScore`.
- Finally, explanatory insights and actionable alerts are emitted to `alerts.jsonl` and rendered on screen.

---

### Diagram 4: PipelineContext State Lifecycle (Blackboard Pattern)

```mermaid
stateDiagram-v2
    [*] --> Initialized: Orchestrator creates session_id
    Initialized --> Ingested: IngestionAgent populates ingestion_metadata
    Ingested --> Harmonized: SchemaAgent populates harmonized_data
    Harmonized --> QualityAssessed: QualityAgent populates quality_metrics
    QualityAssessed --> OperationsEvaluated: OperationalAgent populates operational_kpis
    OperationsEvaluated --> AnomaliesDetected: AnomalyAgent populates anomalies
    AnomaliesDetected --> RiskCalculated: RiskAgent populates risk_scores
    RiskCalculated --> InsightsGenerated: InsightAgent populates insights
    InsightsGenerated --> AlertsCreated: AlertAgent populates alerts
    AlertsCreated --> Completed: Orchestrator timestamps completed_at
    Completed --> [*]
```

#### Diagram 4 Explanation:
- Visualizes the `PipelineContext` as it moves across state transitions.
- Each agent acts as a pure state transform function: $f(\text{Context}) \to \text{Context}'$.
- At no point is state stored in global variables or static singletons, guaranteeing isolation between concurrent sessions.
