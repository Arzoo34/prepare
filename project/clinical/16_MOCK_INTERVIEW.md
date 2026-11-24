# Phase 17: Mock Technical Interview Simulation & Rapid Revision Sheet

This document simulates an intensive, multi-round technical interview for a **Senior Software Engineer / Technical Lead / Data Architect** role, evaluating this exact clinical platform.

---

## Round 1: Project Introduction & High-Level Framing

### Interviewer Question
*"Tell me about this clinical platform you built. What was the motivation and what does it do?"*

### What the Interviewer Is Evaluating
- Ability to synthesize complex business domains into clear, structured engineering summaries.
- Focus on business impact, technical depth, and architectural trade-offs without getting lost in trivial details.

### Strong Sample Answer
> *"In global clinical trials, running multi-site operations costs pharmaceutical sponsors upwards of \$1M a day. A recurring industry bottleneck is that Electronic Data Capture (EDC) systems and lab feeds produce heterogeneous spreadsheets with inconsistent schemas, duplicate subjects, and delayed Serious Adverse Event (SAE) reporting.
>
> To solve this, I designed and built an event-driven multi-agent data intelligence platform in Python. The system deploys 8 specialized agents that safely ingest raw trial workbooks, normalize schemas into Apache Parquet using declarative alias mappings, perform vectorized data quality checks, evaluate operational freshness, detect volume anomalies with sample-bounded z-scores, and compute a composite risk score for each site. An explainability layer translates statistical anomalies into plain-English clinical insights offline or via OpenAI, writing prioritized alerts to an append-only JSONL log and rendering an interactive executive dashboard in Streamlit. The test suite has 100% pass rate across 20 automated tests, and the entire pipeline executes in under 1.5 seconds."*

### Common Weak Answer (What NOT to Say)
> *"I built a Streamlit app that takes Excel files and makes charts. It uses some agents to check if columns are missing and calls ChatGPT to explain the errors. It was for a hackathon."*
- **Why It Fails**: Sounds like a toy project; fails to explain business motivation, architecture, or algorithmic rigor; diminishes the project.

### Follow-up Question
*"What makes this different from setting up an automated dashboard in Tableau or Power BI?"*
- **Strong Follow-up Response**:
  > *"Tableau and Power BI are passive visualizers; they assume data has already been cleaned, normalized, and modeled into a relational schema. If a hospital changes a column name from `Subject ID` to `SUBJ_ID`, a BI dashboard breaks. Furthermore, BI tools cannot perform automated Zip Slip path-traversal validation, cryptographic checksum audit logging, dynamic small-sample z-score bounds, or generate plain-language clinical diagnostic explanations. Our platform is an active, event-driven data processing and intelligence engine that acts as the upstream normalization and risk-scoring layer before visualization."*

---

## Round 2: Architecture & System Boundaries

### Interviewer Question
*"Why did you architect this as a Modular Monolith instead of building 8 independent microservices in Docker containers?"*

### What the Interviewer Is Evaluating
- Pragmatic architectural judgment. Can the candidate resist over-engineering?

### Strong Sample Answer
> *"Microservices solve organizational scaling problems, not performance problems. For this platform, our primary non-functional requirement was fast, deterministic execution on single study archives with zero external infrastructure overhead.
>
> If we decomposed the 8 agents into microservices, we would have introduced massive network serialization latency—passing Pandas DataFrames back and forth over HTTP or gRPC—plus the operational overhead of Docker, Kubernetes, service discovery, and distributed tracing.
>
> Instead, I implemented a **Modular Monolith**: the agents have strict, decoupled boundaries inheriting from `BaseAgent`, share state through a typed `PipelineContext` blackboard, and publish events to an in-memory `EventBus`. This gave us the clean domain boundaries of microservices with the microsecond execution speed and operational simplicity of a single Python process."*

---

## Round 3: Algorithms & Mathematical Rigor

### Interviewer Question
*"You mentioned z-scores for anomaly detection. In small clinical studies with only 3 files, standard z-score thresholds fail mathematically. Did you account for this?"*

### What the Interviewer Is Evaluating
- Real mathematical depth vs. blindly calling `scipy.stats.zscore()`.

### Strong Sample Answer
> *"Yes, exactly. For any sample of size $N$, the maximum theoretical z-score cannot exceed $\sqrt{N-1}$. For a study with only 3 files, the maximum possible z-score is $\sqrt{2} \approx 1.414$. If you blindly set an anomaly threshold of $z \ge 2.5$, it is mathematically impossible to flag an outlier, even if one file contains 10,000 rows while the other two contain 100 rows.
>
> In `AnomalyAgent`, I solved this by engineering a **dual-criterion detection algorithm**: for cohorts with $N \ge 10$, it uses the standard $z \ge 2.5$ rule. For small cohorts ($N < 10$), it applies a sample-bounded threshold of $z \ge 1.3$ combined with a relative variance test ($x > 1.5\mu$ or $x < 0.5\mu$). This ensures that volume explosions are accurately flagged regardless of cohort size."*

---

## Round 4: Security & Compliance Audit

### Interviewer Question
*"Clinical trial systems process patient data and accept ZIP uploads. How do you defend against malicious file uploads and comply with data privacy laws?"*

### What the Interviewer Is Evaluating
- Security consciousness and domain compliance awareness (HIPAA, 21 CFR Part 11).

### Strong Sample Answer
> *"We address security and compliance at three distinct levels:
>
> 1. **Zip Slip Neutralization**: In `IngestionAgent.extract_zip()`, we inspect every archived member before writing to disk. We resolve the absolute canonical target path and assert that it strictly starts with the resolved target directory path string. Any attempt to escape via `../../` throws an immediate `ValueError`.
> 2. **YAML Safe Deserialization**: In `ConfigLoader`, we strictly use `yaml.safe_load(f)` to prevent Remote Code Execution (RCE) via malicious `!!python/object` payloads.
> 3. **Data Privacy (HIPAA / GDPR)**: The pipeline operates 100% offline using our deterministic `LocalInsightGenerator`, ensuring zero patient health information leaves the local environment. When cloud LLMs are enabled, the dataset is pre-anonymized, stripping all direct patient identifiers."*

---

## Round 5: Debugging & Production Incidents

### Interviewer Question
*"Imagine a user uploads an Excel file where every single cell is valid, but the pipeline throws a `ZeroDivisionError` in the risk calculation. Where would that bug originate, and how would you debug it?"*

### What the Interviewer Is Evaluating
- Code tracing ability and edge-case handling.

### Strong Sample Answer
> *"I would immediately trace the quality and risk calculations in `app/agents/quality_agent.py` and `app/agents/risk_agent.py`.
>
> In `QualityAgent`, if a user uploads a valid Excel file that has 0 rows (header only), `df.size` is zero. If you divide `non_null_cells / df.size`, it raises `ZeroDivisionError`. In our code, we explicitly guard against this: `if df.empty or df.size == 0: return 100.0`.
>
> Similarly, in `RiskAgent._calculate_composite_score()`, if no quality metrics were computed because all files were empty, dividing by `len(quality_scores)` would cause division by zero. We handle this with a fallback ternary: `sum(quality_scores) / len(quality_scores) if quality_scores else 50.0`.
>
> To debug this in production, I would inspect `data/logs/orchestrator.jsonl` to identify the `session_id` and examine the `row_count` in `context.ingestion_metadata`."*

---

## 6. Rapid-Fire 60-Second Revision Sheet

| Concept | Key Bullet Point for Rapid Revision |
| :--- | :--- |
| **Architecture Pattern** | Event-Driven Modular Monolith with an 8-Agent Linear Pipeline and Blackboard state. |
| **State Sharing** | Through `PipelineContext` dataclass passed into each agent's `execute(context)` method. |
| **Event Bus** | In-process, synchronous Observer pattern with `publish()`, `subscribe()`, and history retention. |
| **Storage Formats** | Raw spreadsheets in `data/raw/`, Curated tables in Apache Parquet (`pyarrow`), Alerts in `alerts.jsonl`. |
| **Zip Slip Defense** | `member_path.resolve().startswith(extract_to.resolve())` in `IngestionAgent`. |
| **Hashing Algorithm** | Streaming SHA256 in 4KB chunks (`O(1)` constant memory). |
| **Quality Formula** | $(0.50 \times \text{Completeness}) + (0.30 \times \text{Validity}) + (0.20 \times \text{Consistency})$. |
| **Small-Sample Z-Score** | $z \le \sqrt{N-1}$ bound resolved via dual-criterion threshold ($z \ge 1.3$ and $x > 1.5\mu$ for $N < 10$). |
| **Risk Formula** | $(100 - \bar{Q}) \times 0.50 + S_{\text{ops}} \times 0.25 + S_{\text{anom}} \times 0.25$. |
| **LLM Strategy** | `LocalInsightGenerator` (deterministic offline) and `OpenAIInsightGenerator` (with automatic fallback). |
| **Test Suite** | 20 unit/e2e integration tests in `tests/test_platform.py` executing via Pytest in $1.36$ s. |
| **Streamlit State** | Handled via `st.session_state` to prevent re-execution of agents during UI reruns. |
| **Enterprise Evolution** | React SPA + FastAPI + Celery/Temporal workers + AWS S3 + PostgreSQL + Snowflake. |
