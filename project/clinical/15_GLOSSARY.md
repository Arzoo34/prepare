# Phase 16: Comprehensive Clinical & Technical Glossary

---

## 1. Clinical Trial Domain Terminology

- **EDC (Electronic Data Capture)**: Computerized systems (e.g., Medidata Rave, Oracle InForm) designed for the collection of clinical trial data in electronic format from hospital research sites.
- **CPID (Clinical Program Intelligence Data)**: Operational metrics capturing patient recruitment, screening, randomized enrollment, and milestone visit completions across sites.
- **EDRR (Electronic Data Record Review)**: The process of identifying open queries, discrepancy reports, and unverified data points requiring clinical coordinator resolution.
- **SAE (Serious Adverse Event)**: Any untoward medical occurrence that at any dose results in death, is life-threatening, requires inpatient hospitalization, or results in persistent disability. Strictly regulated by FDA/EMA with mandatory 24-hour reporting timelines.
- **MedDRA (Medical Dictionary for Regulatory Activities)**: A standardized international medical terminology dictionary used by regulatory authorities to classify adverse events into hierarchical levels (Lowest Level Term $\to$ Preferred Term $\to$ High Level Term $\to$ System Organ Class).
- **WHODD (World Health Organization Drug Dictionary)**: The international reference dictionary for identifying and categorizing medicinal products and concomitant drugs administered during clinical trials.
- **ICH E6(R2)**: The International Council for Harmonisation's Good Clinical Practice guideline. Revision 2 mandates Risk-Based Monitoring (RBM), requiring sponsors to systematically identify risks to critical data and trial processes.
- **FDA 21 CFR Part 11**: Federal regulations defining the criteria under which electronic records and electronic signatures are considered trustworthy, reliable, and legally equivalent to paper records. Mandates strict audit trails, data integrity, and provenance.
- **CRA (Clinical Research Associate)**: A monitor who oversees trial progress at clinical sites to verify protocol compliance, patient safety, and data completeness.
- **CDM (Clinical Data Manager)**: The specialist responsible for designing data collection forms, issuing data queries, harmonizing schemas, and driving the trial toward Database Lock.
- **DBL (Database Lock)**: The final milestone where clinical study data is frozen, preventing further edits, in preparation for statistical analysis and regulatory submission.
- **SDV (Source Data Verification)**: The process of manually cross-checking data recorded in the EDC against original medical records (hospital charts, lab printouts).

---

## 2. Software Architecture & Engineering Terminology

- **Modular Monolith**: A software architecture pattern where the entire system runs as a single process, but internal code is structured into strictly bounded, decoupled modules with clear public APIs.
- **Agentic Pipeline**: An architectural pattern where specialized functional units ("agents") each encapsulate a bounded business domain, maintain their own rules, and process data sequentially or reactively.
- **Blackboard Architectural Pattern**: A design pattern where a shared computational structure (in our case, `PipelineContext`) is incrementally inspected and enriched by multiple independent subsystems (agents).
- **Observer / Pub-Sub Pattern**: A behavioral pattern where an object (`EventBus`) maintains a list of subscribers and notifies them automatically of state changes by dispatching event objects.
- **Facade Pattern**: A structural pattern providing a simplified unified interface (`PipelineOrchestrator`) over a complex subsystem of underlying classes and agents.
- **Strategy Pattern**: A behavioral pattern enabling an algorithm's behavior to be selected at runtime (`InsightGenerator` implemented as either `LocalInsightGenerator` or `OpenAIInsightGenerator`).
- **Columnar Storage (Apache Parquet)**: A storage format organizing tabular data by column rather than by row. Enables extreme data compression (Snappy, dictionary encoding) and projection pushdown (reading only requested columns from disk).
- **Projection Pushdown**: A database optimization where the query engine reads only the required columns from disk metadata, skipping unneeded columns entirely.
- **Vectorization**: Performing an operation on an entire array of values simultaneously using low-level C SIMD (Single Instruction, Multiple Data) CPU instructions, eliminating slow Python-level interpreted loops.
- **Zip Slip**: A critical security vulnerability where an uncompressed filename inside a ZIP archive contains directory traversal paths (`../../`), allowing an attacker to overwrite arbitrary files on the host filesystem.
- **AST Assertion Rewriting**: A compiler technique used by Pytest where standard Python `assert` statements are parsed into Abstract Syntax Trees and rewritten into rich introspection trees before bytecode execution.
- **Timezone-Aware vs. Naive Datetime**: A timezone-naive datetime lacks timezone information and defaults to local system clocks. A timezone-aware datetime explicitly carries UTC offsets (`datetime.now(timezone.utc)`), preventing ambiguous date math across distributed servers.
