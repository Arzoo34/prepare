# Phase 3C: Architectural Decision Records (ADRs) & Trade-off Analysis

---

## 1. Architectural Decision Records (ADRs)

### ADR-01: In-Process Synchronous EventBus vs. Distributed Message Broker (Kafka / RabbitMQ)

```
┌────────────────────────────────────────────────────────────────────────┐
│ DECISION: Use an in-process, synchronous Python EventBus with history   │
│ retention instead of an external message broker (Kafka, RabbitMQ, SQS). │
└────────────────────────────────────────────────────────────────────────┘
```

- **Problem It Solved**: The system needed a decoupled publish-subscribe mechanism so that agents could publish events (`FILE_INGESTED`, `QUALITY_ASSESSED`, `ALERT_CREATED`) without tight coupling to specific subscribers or logging sinks.
- **Alternatives Considered**:
  1. *Apache Kafka*: High-throughput distributed commit log.
  2. *RabbitMQ / Celery*: AMQP message broker with distributed task workers.
  3. *Redis Pub/Sub*: Lightweight in-memory distributed messaging.
  4. *Direct Method Calls*: Agents directly calling logging and alert functions.
- **Why Alternatives Were Rejected**:
  - Kafka and RabbitMQ introduce significant operational complexity (managing broker nodes, ZooKeeper/KRaft, network partitions, serializing complex objects like DataFrames over the wire).
  - For single-study batch runs, network serialization overhead would make the pipeline 5–10x slower than in-memory object references.
  - Direct method calls would introduce tight coupling, violating the Open/Closed Principle.
- **Advantages of Chosen Approach**:
  - Zero external infrastructure dependencies; runs out-of-the-box in any local environment.
  - Sub-microsecond event delivery latency (in-memory list append and function call dispatch).
  - Preserves event ordering deterministically within the session.
- **Trade-offs Introduced**:
  - Events are ephemeral in memory; if the Python process is abruptly killed (`SIGKILL`), unpersisted event history is lost.
  - Bounded to a single process; cannot fan out events to worker nodes on different physical machines.
- **Tipping Point for Migration**:
  - When the platform needs to process 1,000+ clinical studies continuously in parallel across multiple worker machines.
- **Enterprise-Scale Recommendation**:
  - Use **Apache Kafka** or **AWS SQS/EventBridge** where each agent is an independent microservice container triggered by event topics.

---

### ADR-02: File-Based Parquet & JSONL Storage vs. Relational Database (PostgreSQL / MySQL)

```
┌────────────────────────────────────────────────────────────────────────┐
│ DECISION: Store curated datasets in Apache Parquet files and alerts    │
│ in append-only JSONL files rather than an RDBMS or NoSQL database.     │
└────────────────────────────────────────────────────────────────────────┘
```

- **Problem It Solved**: Needed a persistent storage layer for diverse clinical datasets whose schemas vary wildly across studies (e.g., an SAE table has 25 columns, while an EDC metrics table has 90 columns).
- **Alternatives Considered**:
  1. *Relational Database (PostgreSQL)*: Rigid relational schemas with foreign keys.
  2. *Document Database (MongoDB)*: Flexible JSON document storage.
  3. *Raw CSV Files*: Plain text comma-separated tables.
- **Why Alternatives Were Rejected**:
  - Relational databases require strict upfront schema definitions (`DDL`). Because clinical trial protocols differ, creating tables dynamically for every custom trial workbook leads to severe schema drift and migration nightmares.
  - CSV files do not store column data types, do not support binary compression, and take 10x longer to parse on re-read.
  - MongoDB adds operational overhead and lacks native analytical column-vectorized query optimizations.
- **Advantages of Chosen Approach**:
  - **Columnar Efficiency**: Apache Parquet compresses clinical tabular data by up to **80%** using Snappy/Gzip dictionary encoding.
  - **Schema Preservation**: Parquet stores strongly typed schemas (integers, floats, timestamps) embedded in the file metadata.
  - **Append-Only Auditability**: JSON Lines (`alerts.jsonl`) allows atomic appending of log records without locking entire database tables.
- **Trade-offs Introduced**:
  - No native ACID transaction support across multiple files.
  - No SQL indexing or cross-dataset `JOIN` operations without reading files into memory.
- **Tipping Point for Migration**:
  - When concurrent web users need complex relational queries (e.g., "Find all subjects across 200 trials who had nausea within 3 days of dose").
- **Enterprise-Scale Recommendation**:
  - Store raw Parquet in an object store (**AWS S3 / Azure Data Lake Storage**) and query them through a modern Lakehouse engine like **Databricks Delta Lake** or **Snowflake**.

---

### ADR-03: Vectorized Pandas/NumPy vs. Distributed PySpark Pipeline

```
┌────────────────────────────────────────────────────────────────────────┐
│ DECISION: Build the core multi-agent engine using vectorized Pandas    │
│ and NumPy rather than executing PySpark natively for single-trial runs.│
└────────────────────────────────────────────────────────────────────────┘
```

- **Problem It Solved**: The repository initially contained an experimental PySpark pipeline (`src/pipeline.py`), which required Java Virtual Machine (JVM) runtimes, winutils binary setup on Windows, and heavy memory overhead.
- **Alternatives Considered**:
  1. *Pure PySpark*: Distributed DataFrame processing.
  2. *DuckDB*: In-process SQL OLAP query engine.
  3. *Polars*: Rust-backed multi-threaded DataFrame library.
- **Why PySpark Was De-emphasized for Core Agents**:
  - PySpark has a **3–8 second JVM initialization overhead** before executing a single line of code. For small-to-medium clinical studies (1,000–50,000 rows), PySpark is significantly slower than Pandas due to JVM inter-process communication (Py4J).
  - PySpark requires heavy external binary dependencies on Windows (`winutils.exe`, `HADOOP_HOME`, Java 8/11/17).
- **Advantages of Chosen Approach**:
  - Executes instant calculations ($< 1.5$ seconds for full pipeline runs).
  - Direct zero-copy memory integration with Plotly and Streamlit.
  - Simple local debugging with standard Python breakpoints.
- **Trade-offs Introduced**:
  - Constrained to single-node memory (RAM). Datasets larger than 8–16 GB cannot fit into memory simultaneously on standard workstations.
- **Enterprise-Scale Recommendation**:
  - Use **DuckDB** or **Polars** for medium datasets on single hosts (up to 100M rows), and invoke **Databricks Spark** only for multi-terabyte cross-trial epidemiological aggregations.

---

### ADR-04: Deterministic Local Templating vs. Direct Cloud LLM API Reliance

```
┌────────────────────────────────────────────────────────────────────────┐
│ DECISION: Abstract insight generation behind a Strategy Pattern with a │
│ default deterministic LocalInsightGenerator and optional OpenAI client.│
└────────────────────────────────────────────────────────────────────────┘
```

- **Problem It Solved**: AI explanation is needed to translate statistical alerts into human prose, but pharmaceutical trials operate under strict data privacy regulations (HIPAA, GDPR) and cannot mandate active internet connections or third-party cloud API tokens.
- **Alternatives Considered**:
  1. *Mandatory OpenAI API Call*: Always sending prompt text to `api.openai.com`.
  2. *Local Quantized Model (Ollama / Llama.cpp)*: Running an 8B model locally.
  3. *Static Hardcoded Text Strings*: Generic non-interpolated error messages.
- **Why Alternatives Were Rejected**:
  - Mandatory cloud APIs introduce billing costs, rate limits, network failure points, and severe PHI compliance risks.
  - Local LLMs (Ollama) require 4–8 GB of dedicated GPU VRAM and complex C++ runtime installations.
- **Advantages of Chosen Approach**:
  - `LocalInsightGenerator` is 100% deterministic, offline, zero-latency, and requires zero API keys.
  - Seamlessly swaps to `OpenAIInsightGenerator` if an API key is provided, maintaining the exact same interface.
- **Trade-offs Introduced**:
  - Local template rules cannot answer free-form conversational follow-up questions from the user.
- **Enterprise-Scale Recommendation**:
  - Deploy a private self-hosted, HIPAA-compliant open-weights model (e.g., Llama-3-Med-8B or Mistral) on a dedicated VPC endpoint behind an internal enterprise API gateway.

---

### ADR-05: Linear Synchronous Pipeline vs. Autonomous Asynchronous Agents

```
┌────────────────────────────────────────────────────────────────────────┐
│ DECISION: Orchestrate agents in a deterministic synchronous sequence   │
│ via PipelineOrchestrator rather than fully autonomous async agents.    │
└────────────────────────────────────────────────────────────────────────┘
```

- **Problem It Solved**: How should 8 agents coordinate their execution and share state without creating race conditions or nondeterministic loops?
- **Alternatives Considered**:
  1. *Autonomous Agent Loops (LangChain / AutoGen / CrewAI)*: LLM decides which agent to call next based on conversational prompts.
  2. *Asynchronous Event Loop (asyncio)*: Concurrent agent execution with locks.
  3. *Directed Acyclic Graph (DAG) Engine (Airflow / Prefect)*.
- **Why Autonomous Agent Loops Were Rejected**:
  - Autonomous LLM loops are non-deterministic, prone to infinite loops, expensive in token consumption, and fail regulatory audits for validated systems. In a clinical trial, schema harmonization MUST precede quality monitoring, which MUST precede risk scoring.
- **Advantages of Chosen Approach**:
  - 100% deterministic, repeatable, and easily testable execution order.
  - Zero concurrency bugs (no deadlocks, race conditions, or lock contention).
- **Trade-offs Introduced**:
  - Independent stages (e.g., Quality vs. Operations) run sequentially rather than concurrently, slightly increasing total execution time.
- **Enterprise-Scale Recommendation**:
  - Model the pipeline as an **Apache Airflow or Temporal DAG** where independent tasks run on parallel workers with strict dependency validation.

---

## 2. Architecture Comparison Matrix

| Evaluation Dimension | Our Chosen Architecture | Microservices + Kafka | Pure PySpark Cluster | Autonomous LangChain/CrewAI |
| :--- | :--- | :--- | :--- | :--- |
| **Deployment Complexity** | **Very Low** (Single `pip install`) | **Very High** (K8s, Broker, Service Mesh) | **High** (Cluster, JVM, Spark Master/Workers) | **Medium** (Requires LLM API keys & orchestration) |
| **Execution Latency** | **Sub-second** ($1.3 - 2.5$ s) | Low to Medium (Network hop overhead) | Medium ($5 - 15$ s minimum startup) | Very Slow ($10 - 60$ s per agent turn) |
| **Regulatory Determinism** | **100% Deterministic** | 100% Deterministic | 100% Deterministic | **Low / Nondeterministic** |
| **Max Scalability (Data)** | Single Node ($\approx 5$ GB RAM) | Highly Distributed (Petabytes) | Highly Distributed (Petabytes) | Single Node (Token context limited) |
| **Operational Cost** | **\$0 (Free / Open Source)** | High (Cloud infrastructure & hosting) | Medium-High (Compute clusters) | High (Token billing per run) |
| **Debugging Simplicity** | **High** (Standard Python debugger) | Hard (Distributed tracing required) | Medium (JVM stack traces) | Very Hard (Prompt fragility) |
| **Suitability for This App** | **Ideal for targeted clinical review** | Overkill for single studies | Overkill for sub-100K row studies | Unsafe for regulatory compliance |
