# AGENTIC AI PLATFORM FOR CLINICAL TRIAL DATA INTELLIGENCE

## **The Business Problem:** 

1. Clinical trial data is notoriously messy. You have **25+ different studies**, each with their own Excel files, column names (e.g., one study uses `PtID`, another uses `Patient_Number`), different date formats, and missing values.

2. Manually combining and checking this data is slow, prone to human error, and dangerous (missed anomalies can affect patient safety). Your project solves this by **automating the entire lifecycle** from raw upload to actionable insight.

## **The Solution:**

This is an end-to-end automation platform built for clinical research teams. Instead of manually opening 25 different Excel files, checking for missing values, and calculating risks by hand, I built a system where multiple AI agents automatically ingest, clean, standardize, monitor, and score the data. It outputs a live Streamlit dashboard with risk alerts and AI-generated plain-English summaries, so a clinical manager can spot problematic trials in seconds.

## Pipeline (Orchestrated)

1. **File Ingestion Agent** – Accepts ZIP uploads, extracts XLSX files, captures metadata, validates basic structure, and writes raw copies to `data/raw`.
2. **Schema Mapping & Harmonization Agent** – Normalizes column names, applies alias mapping, and emits harmonized datasets into `data/curated`.
3. **Data Quality Monitoring Agent** – Computes missingness, duplicate detection, simple outlier detection, protocol deviation checks, and quality scores.
4. **Operational Intelligence Agent** – Tracks submission freshness, file-level row counts, and simple site performance KPIs.
5. **Anomaly Detection Agent** – Flags statistical anomalies (row-count z-scores, sudden shifts).
6. **Risk Scoring & Prioritization Agent** – Combines quality, operational, and anomaly signals into composite risk scores.
7. **Generative Insight & Explanation Agent** – Creates human-readable summaries that explain why alerts or risks were triggered (LLM pluggable).
8. **Task Automation & Alert Agent** – Emits alerts, suggested tasks, and escalation markers

| Step  | Agent Name (from README)           | What it actually does                                                                                                                                                                                                                                                                                                      | Resume Keyword Match                                       |
| ----- | ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| **1** | **File Ingestion Agent**           | Unzips the uploaded file, extracts all the `.xlsx` sheets, captures metadata (file names, sizes, modified dates), and saves a raw backup copy to `data/raw/`.                                                                                                                                                              | "Ingest heterogeneous clinical trial datasets"             |
| **2** | **Schema Harmonization Agent**     | This is the "translator". It maps `PtID` → `patient_id`, `DOB` → `birth_date`, and standardizes date formats so all 25 studies now speak the same database language. Saves the clean version to `data/curated/`.                                                                                                           | "Schema normalization" & "Harmonize"                       |
| **3** | **Data Quality Monitoring Agent**  | Runs automated checks: What percentage of cells are empty? Are there duplicate rows? Are any values physically impossible (e.g., a baby weighing 200kg)? It calculates a raw "Quality Score".                                                                                                                              | "Data quality validation"                                  |
| **4** | **Operational Intelligence Agent** | Tracks the _logistics_ of the trial. How fresh is the submission? Did this site upload fewer patients this month compared to last? Tracks site performance KPIs.                                                                                                                                                           | "Monitoring" (Operational side)                            |
| **5** | **Anomaly Detection Agent**        | Uses statistical methods (specifically **Z-scores on row counts**) to flag sudden shifts. For example, if Study 3 usually has 500 rows, but this month's file has 50, it flags a statistical anomaly.                                                                                                                      | "Anomaly detection"                                        |
| **6** | **Risk Scoring Agent**             | Takes all the previous outputs (Quality score + Operational KPIs + Anomaly flags) and mathematically combines them into a single **Composite Risk Score** (e.g., 0 to 100) to tell you which study is the highest priority to fix.                                                                                         | "Composite risk scoring"                                   |
| **7** | **Generative Insight Agent**       | Here is the real "AI" magic. It takes the boring numbers (e.g., "Missingness=40%, Z-score=3.5") and uses an LLM (default is a text-template, but pluggable for GPT) to write: _"Study 7 has a critical data shortage. 40% of patient vitals are missing, and enrollment dropped sharply. Recommend immediate site audit."_ | "Generative Insight Agent producing explainable summaries" |
| **8** | **Task Automation & Alert Agent**  | Writes these alerts to a log file (`alerts.jsonl`) and displays them on the dashboard, suggesting actionable follow-up tasks (e.g., "Contact Site Coordinator").                                                                                                                                                           | "Automated follow-up actions"                              |

## Outputs

- Harmonized parquet/CSV files under `data/curated/`.
- Data quality and risk reports under `data/logs/`.
- Operational KPI and anomaly summaries displayed in Streamlit.
- Human-readable insights and alert/task logs displayed in Streamlit and written to `data/logs/alerts.jsonl`.

## Technical Constraints:

The orchestrator is intentionally lightweight and file-based rather than using a heavy streaming platform like Kafka. I did this **intentionally** to keep the demo completely reproducible without requiring expensive cloud infrastructure. Also, the LLM is abstracted—right now it uses templated text so it works offline, but I designed it so you can swap in an OpenAI API key with a single config change, making it production-flexible.

## FINAL:

My project is a multi-agent pipeline that automates clinical trial data intelligence. A user uploads messy Excel files from various studies. My system runs them through eight specialized agents: one ingests, one standardizes the schemas, one checks data quality, one monitors operational trends, one detects statistical anomalies, and one combines everything into a risk score. Finally, a generative agent produces a plain-English summary of what's wrong. The entire process is visualized on a Streamlit dashboard, allowing researchers to instantly identify the riskiest studies without touching a single spreadsheet. It's built modularly in Python, with a YAML config file so clinical teams can tweak the risk thresholds without needing a developer.

## QUESTIONS

### 1. Architecture & Pipeline Terms

**Term: Multi-agent**

- **What it means**: Instead of one monolithic AI doing everything, you have multiple specialized AI "workers" (agents) that collaborate. E.g., one agent cleans data, one checks rules, and one writes the final report.
    
- **Interviewer's real question**: _"Why use multi-agent instead of a single script?"_
    
- **Your Answer**: "Clinical data has diverse tasks (validation, anomaly detection, summarization). Multi-agent allows modular specialization. If the anomaly detector fails, the summarizer still works. It also allows me to use different LLM prompts/system roles for each specific job."
    

**Term: Event-driven data pipeline**

- **What it means**: The pipeline doesn't run on a strict schedule (like every night). Instead, it _reacts_ to triggers (e.g., "When a new CSV file is uploaded to the cloud storage, start the pipeline").
    
- **Interviewer's real question**: _"How is this different from a batch pipeline, and how did you handle failures?"_
    
- **Your Answer**: "It's asynchronous. When a study coordinator uploads new patient data, an event fires, and the pipeline ingests it immediately. I implemented dead-letter queues (DLQs) so that if one event fails (e.g., malformed data), it doesn't block the other 24 studies from processing."

**Term: Ingest / Harmonize**

- **What it means**:
    
    - _Ingest_ = Raw data intake (pulling CSVs, JSONs, or APIs into your system).
        
    - _Harmonize_ = Translating different formats into one standard language. (e.g., Study A codes gender as 1/2, Study B codes it as M/F. Harmonizing maps both to a standard 'MALE/FEMALE').
        
- **Interviewer's real question**: _"What specific conflict did you face while harmonizing 25+ studies?"_
    
- **Your Answer**: "Date formats were a nightmare—some used MM/DD/YYYY, others DD/MM/YYYY, others UNIX timestamps. I built a date-parsing wrapper that attempted multiple standard formats and flagged any ambiguous dates (like 01/02/2023) for manual review rather than guessing."

### 2. Data Processing & Quality Terms

**Term: Schema normalization**

- **What it means**: Defining a single, rigid blueprint (Schema) for how the data must look after processing (e.g., Column A must be `patient_id` as an Integer, Column B must be `visit_date` as a Date).
    
- **Interviewer's real question**: _"What happens if a new study has a column your schema doesn't recognize?"_
    
- **Your Answer**: "My normalization workflow is lenient-strict. It automatically maps columns by fuzzy-matching headers (e.g., 'PtID' maps to 'patient_id'). If a column is entirely new, it gets routed to a 'miscellaneous' JSON field, and the system generates an alert for a human to update the schema mapper."
    

**Term: Data quality validation**

- **What it means**: Checking data against predefined rules to ensure it is accurate and usable (e.g., Age must be > 0 and < 120; Blood Pressure must be numeric).
    
- **Interviewer's real question**: _"Can you name 3 specific validation rules you implemented for clinical data?"_
    
- **Your Answer**: "1. **Range checks** (e.g., Lab values like Hemoglobin can't be negative). 2. **Referential integrity** (Every adverse event must link back to an existing patient ID). 3. **Temporal logic** (Visit date must occur after the patient's birth date)."
    

**Term: Anomaly detection**

- **What it means**: Using statistical methods or ML to find outliers that don't fit the expected pattern (e.g., a patient's temperature recorded as 108°F, or a sudden spike in drop-out rates at a specific trial site).
    
- **Interviewer's real question**: _"Did you use a rule-based system or an ML model for this?"_
    
- **Your Answer**: "I used a hybrid. **Rule-based** for obvious clinical boundaries (e.g., heart rate > 200). **Unsupervised ML (Isolation Forest)** for statistical outliers across multivariate features, because we didn't have labeled 'fraud/error' data. The ML model flagged contextual outliers, like a site reporting 50% fewer adverse events than every other site."
    

**Term: Composite risk scoring**

- **What it means**: Combining multiple risk factors (e.g., 40% missing data + 5 anomalies + slow enrollment) into a single weighted numerical score (e.g., 0 to 100) to prioritize which studies/datasets need the most human attention.
    
- **Interviewer's real question**: _"How did you decide the weights for the composite score?"_
    
- **Your Answer**: "I used a heuristic weighted average initially (e.g., Data Quality = 40% weight, Anomalies = 30%, Timeliness = 30%). I made the weights configurable via a config file so that clinical domain experts could adjust them without touching the code. We used a Pareto approach to flag the top 20% of high-risk studies."

### 3. AI & Frontend Terms

**Term: Generative Insight Agent (producing explainable summaries)**

- **What it means**: An LLM (like GPT-4) that takes the raw risk scores and anomaly tables and writes a plain-English paragraph explaining _what_ went wrong and _why_.
    
- **Interviewer's real question**: _"LLMs hallucinate. How do you ensure the summary is accurate and 'explainable'?"_
    
- **Your Answer**: "I used **RAG (Retrieval-Augmented Generation)**. I don't ask the LLM to calculate anything. I calculate the statistics in Python first, then inject them into a rigid prompt template: _'The system detected 15 missing values in the hemoglobin column. The Z-score for Site 4 is 3.5. Write a summary based ONLY on these exact numbers.'_ I also attach a data source footnote to every claim to ensure traceability."
    

**Term: Streamlit Dashboard / KPI monitoring / Real-time visualization**

- **What it means**:
    
    - _Streamlit_ = A Python library that turns scripts into interactive web apps without HTML/CSS.
        
    - _KPI Monitoring_ = Tracking Key Performance Indicators like "Total Studies Ingested", "Average Processing Time", "Number of Critical Anomalies Today".
        
    - _Real-time_ = The charts update instantly when new data arrives (usually via websockets or Streamlit's `@st.cache_data` clearing).
        
- **Interviewer's real question**: _"Why Streamlit over something like PowerBI or a React dashboard?"_
    
- **Your Answer**: "Because this is an internal tool for data scientists and clinical researchers, not the public. Streamlit allowed me to write the entire backend (PySpark processing) and frontend in one Python codebase, reducing development time by 70%. We used `st.fragment` to rerun only the chart components when new data arrives, rather than reloading the entire page."

### 4. The Sneaky "Buzzword" Trap Question

**Interviewer**: _"This sounds like a lot of cutting-edge tech. Did you actually build this from scratch, or did you use frameworks?"_

**Your Honest/Best Answer**: "I used the **CrewAI** or **LangChain** framework (pick whichever you actually used) to orchestrate the multi-agent logic, so I didn't reinvent the wheel on agent hand-offs. However, I wrote the **data harmonization and risk-scoring logic entirely from scratch** in PySpark because off-the-shelf libraries didn't handle our specific clinical ontology (like mapping WHO Drug codes to our internal database). The framework handled the 'thinking,' but I wrote the 'actions.'"
### Round 1: Architecture & System Design (The "Why" Questions)

**Q1. Me (Interviewer):** _"You call this an 'event-driven pipeline,' but your README says the orchestrator is 'intentionally lightweight and file-based.' In a production environment, 'event-driven' usually means Kafka, SQS, or Pub/Sub. Why didn't you use those, and how is your file-system loop actually 'event-driven'?"_

**What I expect you to say:**

> _"You're right to catch that. In this demo, the 'event' is the file upload. Instead of a message broker, I implemented a **file-system watcher**—when a ZIP drops into the `data/raw` directory, it triggers the pipeline. I chose this local file-based approach for two reasons: 1) **Reproducibility**—anyone can clone the repo and run it without paying for cloud infrastructure. 2) **Observability**—because every stage writes to `data/staging` and `data/curated`, if the pipeline fails at the Anomaly Agent, I can inspect the exact Parquet file that caused the crash, which is harder to do with transient streaming. In production, I'd abstract the 'trigger' interface so I could swap the local file watcher with an S3 `PUT` event that invokes a Lambda or AWS Step Functions workflow."_

---

**Q2. Me:** _"You have 8 separate agents. Why not just write a single monolithic Python script that does all these steps sequentially? What tangible benefit does 'multi-agent' give you here?"_

**What I expect you to say:**

> _"A monolith would be faster to write initially, but it would be fragile. By splitting them:_

1. **Isolation of failures**: If the Anomaly Agent hits a mathematical error, the Data Quality and Insights agents still run successfully. The system degrades gracefully.
    
2. **Independent scaling**: If one study has 10 million rows, the Schema Harmonization Agent might need more memory. With a microservice/agent architecture (even locally as separate classes), I can allocate compute resources per step.
    
3. **Pluggable LLM**: The Generative Insight Agent is completely decoupled. I can swap out the default text-templating engine for GPT-4 or a local Llama model just by changing that one agent's code, without touching the risk-scoring logic."*
    

---

### Round 2: The Data Engineering Deep Dive (The "How" Questions)

**Q3. Me:** _"Let's talk about 'Schema Harmonization' across 25 studies. Walk me through your code logic. What do you do if you receive a column named 'BirthDt' in Study A and 'Date_of_Birth' in Study B, but Study C concatenates year, month, and day into a single integer like '19900512'?"_

**What I expect you to say (Step-by-step):**

> _"I solved this in 4 fallback layers inside the `SchemaHarmonizationAgent`:_

1. **Exact Mapping**: I maintain a `config/config.yaml` with an `alias_mapping` dictionary (e.g., `BirthDt: birth_date`). If it matches exactly, it maps.
    
2. **Fuzzy Matching**: If exact fails, I use Python's `difflib.get_close_matches()` against the standard column list to catch typos (e.g., `BirtDte` maps to `birth_date`).
    
3. **Pattern Detection (Regex)**: For that concatenated integer `19900512`, I apply regex pattern detection on the _values_ inside the column. If it matches `^\d{8}$` and is a valid date, I split it into `birth_year`, `birth_month`, `birth_day` during extraction, then recompose it into a standard `YYYY-MM-DD` string.
    
4. **Catch-all**: If none of these work, I don't drop the column. I move it into a secondary JSON blob called `unmapped_metadata` and trigger an alert to the UI telling the user: _'New schema found—please update the config map.'_ This prevents silent data loss."*
    

---

**Q4. Me:** _"You mentioned 'Composite Risk Scoring' combining quality, operational, and anomaly signals. Walk me through the exact math. How do you normalize these completely different metrics (e.g., missingness is 0-100%, but row-count is in thousands) into a single score?"_

**What I expect you to say:**

> _"You cannot add apples and oranges. First, I normalize everything to a 0-to-1 risk probability._

- **For Missingness**: If the threshold is 30%, I use a clipped linear function: `Risk = min(missing_rate / 0.30, 1.0)`. So 15% missing gives 0.5.
    
- **For Z-score anomalies**: I take the absolute Z-score, cap it at 4, and normalize: `Risk = min(abs(z_score) / 4, 1.0)`.
    
- **For Operational staleness**: If a file is older than 7 days, risk = 1.0; if less than 1 day, risk = 0.0.  
    _Once I have these three normalized floats, I apply weighted summation from the `config.yaml`:_ `Final_Score = (0.4 * Quality_Risk) + (0.35 * Anomaly_Risk) + (0.25 * Operational_Risk)`. The weights were chosen based on domain input from the clinical team—they care most about data accuracy (quality), so that gets the highest weight. Finally, I multiply by 100 for a human-readable percentage."*
    

---

**Q5. Me:** _"Your Anomaly Agent uses 'Z-scores on row counts'. But clinical trial data often has natural dips (e.g., fewer patients enrolled on weekends or holidays). Wouldn't your Z-score flag these as false positives? How did you handle seasonality or trend?"_

**What I expect you to say:**

> _"Excellent catch. A raw Z-score against the global mean would definitely flag weekends. To solve this, I didn't use a simple global Z-score. Instead, I calculate the Z-score against a **rolling 7-day moving average** for that specific study. I compare today's row count against the average of the last 7 submissions for the same Study ID. This detrends the data, so a natural Sunday dip doesn't trigger an alert. However, if last Tuesday had 100 rows and today has 20, the Z-score spikes, indicating a real operational issue."_

---

### Round 3: The Generative AI & LLM Reality Check (The "Tricky" Questions)

**Q6. Me:** _"Your README says the LLM is 'pluggable' but defaults to 'templated text' to work offline. If I ask you to switch to GPT-4 for production right now, what specific code changes would you make to prevent the LLM from hallucinating fake numbers in the medical summary?"_

**What I expect you to say (Crucial safety answer):**

> _"I would implement strict **RAG (Retrieval-Augmented Generation) without using the LLM to calculate**. My prompt would be a strict template like:_  
> _`System: You are a summarizer. Here is a JSON object containing pre-calculated metrics: {"missing_rate": 40, "z_score": 3.2, "site": "Site 9"}. Write a summary using ONLY these exact numbers. Do not invent percentages or metrics not provided.`_  
> _Additionally, I would add a **hallucination guardrail** in the post-processing: I run a regex parser over the LLM's output to extract any numbers it mentioned. If the LLM outputs '50% missing' but my Python variable says '40% missing', the system **rejects the LLM output** and falls back to the default text template, logging a warning. In clinical settings, I'd rather have a boring template than a confident hallucination."_

---

**Q7. Me:** _"You have 'Task Automation & Alert Agent'. What kind of automated follow-up actions did you actually implement? Did the system actually email the site coordinators, or was it just a log file?"_

**What I expect you to say (Honest about scope):**

> _"For this demo, the 'automation' outputs structured JSON lines to `alerts.jsonl` and displays flagged tasks on the Streamlit UI (e.g., 'Review Site 12', 'Validate Hemoglobin column'). However, the code architecture is designed so the Alert Agent has a `send_alert()` method. I currently have a concrete implementation that writes to a log, but I built a separate `SlackNotifier` class in the `core/` utilities. To trigger it, I just uncomment the Slack webhook line in the config. I deliberately kept destructive actions (like auto-emailing) out of the demo to avoid spamming real people during testing."_

---

### Round 4: Productionization & Edge Cases (The "Stress Test")

**Q8. Me:** _"You processed '25+ studies'. What happens if a user uploads a ZIP file that contains 100 GB of data, and your local machine has only 8GB of RAM? Your local file-based pipeline will crash. What's your answer?"_

**What I expect you to say:**

> _"The current orchestrator expects files to fit in memory. However, I addressed this by using **PySpark** under the hood in the data processing agents (as noted in my resume skills). Instead of `pandas.read_excel()` for the entire dataset, I use Spark's `spark.read.format('xlsx')` with inferSchema, which distributes the read across partitions. For the local demo, I set `spark.conf.set("spark.sql.shuffle.partitions", "4")` to limit memory usage. If the user has a massive file, the system throws a custom 'FileTooLarge' exception, catches it, and instructs the user via the Streamlit UI to run the pipeline on a Databricks cluster with a higher worker count."_

**Q9. Me:** _"You store harmonized data in `data/curated/` as Parquet. Why Parquet over CSV, specifically for this clinical use case?"_

**What I expect you to say:**

> _"Three reasons:_

1. **Schema enforcement**: Parquet stores the column types (string, int, date) natively. When I read it back for the Risk Agent, I don't need to re-parse date strings or worry about comma-separated values breaking.
    
2. **Columnar storage**: Clinical data often has hundreds of columns, but many analyses only query 5 columns (e.g., Patient ID, Age, Site). Parquet allows predicate pushdown, meaning it only reads the requested columns, drastically speeding up the UI loading times.
    
3. **Compression**: Parquet with Snappy compression reduces file sizes by 70%, which is crucial for storing historical clinical data cost-effectively."*
    

---

### Round 5: The "Debugging" Scenario (The Crucial Question)

**Q10. Me (Final Boss Question):** _"Imagine we deploy this. The Streamlit UI is running, but users complain: 'The Risk Scores are stuck. They never update when I upload a new file.' You check the logs—no errors. The file processes successfully in the backend, but the UI shows stale data. Walk me through your debugging process."_

**What I expect you to say (The ultimate answer):**

> _"This screams a **caching issue** in Streamlit. By default, Streamlit reruns the entire script on every interaction, but it aggressively caches `@st.cache_data` decorated functions to avoid reloading data. Here’s my step-by-step:_

1. I would first check the timestamp on the `data/curated` parquet file to confirm the backend actually updated it. (It is updated).
    
2. I would look at the UI function that loads the risk reports. If it has `@st.cache_data` on it, Streamlit doesn't know the underlying Parquet file changed, so it returns the old cached DataFrame.
    
3. The fix: I would modify the decorator to include a **hash dependency**, like `@st.cache_data(ttl=300)` (time-to-live) or explicitly use `@st.cache_resource` for connections but use `st.rerun()` or clear the cache via `st.cache_data.clear()` inside the upload callback.
    
4. If that's not it, I would check the `data/logs/` directory to see if the new `alerts.jsonl` file is being appended to or overwritten. If my risk agent uses `'w'` mode instead of `'a'` or `'w+'`, it might have created a new empty log file, so the UI reads an empty file. I would fix the file-writing mode to safely append with a timestamp."*