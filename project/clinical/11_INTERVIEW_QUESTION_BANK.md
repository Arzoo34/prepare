# Phase 12: Complete Technical Interview Question Bank (Levels 1–13)

This question bank contains over 80 targeted technical interview questions designed to test deep code comprehension, architectural reasoning, algorithmic fluency, and production readiness based strictly on this codebase.

---

## Level 1: Project Fundamentals

1. What does this project do in 60 seconds or less?
2. What specific pain points in clinical trial data management does this platform address?
3. What is the regulatory context (e.g., ICH E6(R2), FDA 21 CFR Part 11) that makes automated data quality monitoring mandatory?
4. Who are the primary end-users of this platform, and how does each user interact with the dashboard?
5. Why is a multi-agent pipeline better suited for this problem than a static monolithic SQL dashboard?
6. What is the scope of the project, and what features are intentionally out of scope?

---

## Level 2: Deep Code-Level Understanding

7. Walk me through the class hierarchy of the agents. What methods does `BaseAgent` provide?
8. In `app/orchestrator.py`, how is state shared between agents? Why did you use `PipelineContext` instead of function returns?
9. In `app/agents/ingestion_agent.py`, how does `extract_zip()` prevent path traversal attacks?
10. How does `IngestionAgent._calculate_checksum()` achieve $O(1)$ memory usage when hashing a 500MB Excel file?
11. In `app/agents/schema_agent.py`, explain the normalization algorithm. What happens if a column name matches neither an exact alias nor a space-stripped alias?
12. In `app/agents/quality_agent.py`, how are completeness, validity, and consistency calculated mathematically?
13. In `app/agents/operational_agent.py`, how is file freshness evaluated, and what constitutes a "critical" freshness status?
14. In `app/agents/anomaly_agent.py`, why did you implement a dual-criterion z-score calculation instead of a simple static threshold?
15. In `app/agents/risk_agent.py`, walk me through the exact formula used to calculate composite risk scores.
16. In `app/agents/alert_agent.py`, how are alerts persisted to disk, and why is `append_jsonl` used instead of rewriting a single JSON file?
17. In `app/llm/insight.py`, how does `OpenAIInsightGenerator` support both modern OpenAI v1+ and legacy v0 APIs?

---

## Level 3: Technology Stack & Internals

18. Why did you choose Python 3.10+ over Go or Java for this system?
19. How does Python's Global Interpreter Lock (GIL) impact the performance of our multi-agent pipeline?
20. Explain the internal memory difference between loading an Excel file via `openpyxl` vs. storing it as a Pandas DataFrame.
21. What is Apache Parquet, how does it store data internally, and why did you choose it over CSV?
22. How does `pyyaml.safe_load()` differ from `pyyaml.load()`, and what severe security risk does it mitigate?
23. How does Streamlit's reactive execution model work, and why is `st.session_state` necessary in `streamlit_app.py`?
24. How does Plotly render interactive charts inside the browser?
25. How does Pytest's AST assertion rewriting work internally compared to standard Python `assert`?

---

## Level 4: Architecture & Design Patterns

26. Why is this system described as an "Event-Driven Modular Monolith"?
27. Where is the **Facade Pattern** used in the codebase, and what problem does it solve?
28. Where is the **Strategy Pattern** used in the codebase, and how does it enable offline operation?
29. Where is the **Observer / Publish-Subscribe Pattern** used, and what are its decoupling benefits?
30. Where is the **Template Method Pattern** used in the agent hierarchy?
31. How does the **Blackboard Pattern** apply to `PipelineContext`?
32. What are the module boundaries between `app/core`, `app/agents`, and the presentation layer?
33. Why did you not build this as 8 independent microservices running in Docker containers?

---

## Level 5: Technical Decision-Making & Trade-Offs

34. Why did you choose an in-process `EventBus` over Apache Kafka or RabbitMQ?
35. What are the trade-offs of using file-based Parquet/JSONL storage instead of a relational database like PostgreSQL?
36. The repository has legacy PySpark code in `src/`. Why did you transition the active pipeline to Pandas/PyArrow?
37. Why did you use local deterministic templating as the default insight generator rather than requiring an external LLM API?
38. Why did you choose a sequential orchestrator instead of letting agents run fully autonomously via LLM tool-calling?
39. When would the decision to use Streamlit become a bottleneck, and what would you replace it with?

---

## Level 6: Algorithms, Math & Complexity

40. What is the time and space complexity of the entire pipeline when processing $F$ files with $R$ rows and $C$ columns?
41. Prove mathematically why a sample of size $N=3$ cannot exceed a z-score of $\sqrt{2} \approx 1.414$.
42. How does `pandas.DataFrame.duplicated()` work under the hood, and what is its computational complexity?
43. How does `ConfigLoader.get()` achieve hierarchical key traversal, and what is its time complexity?
44. Explain the clamping and normalization logic in `RiskAgent._calculate_composite_score()`.
45. How would you optimize the outlier detection algorithm if numeric columns contained 50 million rows?

---

## Level 7: Data Integrity & Storage

46. What specific clinical data formats are ingested (EDC, EDRR, SAE, MedDRA, WHODD), and what does each represent?
47. How does the storage layer partition curated Parquet files by study, and what is the query benefit?
48. What happens if an ingested Excel sheet has duplicate column names (e.g., two columns named `Date`)?
49. Why is JSON Lines (`.jsonl`) preferred over standard JSON for streaming event and alert logs?
50. What is the role of the legacy Flask REST API in `app/main.py`, and what endpoints does it expose?

---

## Level 8: Security Audit & Compliance

51. How does `IngestionAgent` defend against the Zip Slip vulnerability? Walk me through the exact path check.
52. What is CSV / Formula Injection, and why does saving data as Parquet protect against it?
53. How does this platform comply with HIPAA and GDPR data privacy standards regarding patient data?
54. If an investigator uploads a 10GB "decompression bomb" ZIP file, what happens, and how would you mitigate it?
55. How are secrets and API keys managed in this project? Are any tokens committed to git?

---

## Level 9: Performance & Scalability

56. What is the peak RAM consumption of the platform during a typical 10-workbook study run?
57. What happens if 100 users hit the Streamlit dashboard simultaneously? What breaks first?
58. What happens if a single study file contains 20 million rows? Which agent fails first with an OOM error?
59. How would you redesign this architecture to handle 10,000 active clinical trials running concurrently?
60. Walk me through the enterprise migration path to AWS (S3, EKS, Celery, PostgreSQL, Snowflake).

---

## Level 10: Debugging & Troubleshooting Scenarios

61. You run `pytest` on Windows and get `ImportError: DLL load failed while importing timestamps: An Application Control policy has blocked this file`. What caused this and how do you resolve it?
62. A hospital uploads a file where `row_count` is 10,000, while the historical mean is 2,000. Why did `AnomalyAgent` NOT flag it if $N=2$?
63. A user reports that the Streamlit dashboard resets every time they click an alert card. What is causing this?
64. An uploaded spreadsheet causes `SchemaAgent` to throw an unhandled exception. How does the orchestrator prevent the entire pipeline from crashing?
65. The OpenAI API starts returning `HTTP 429 Too Many Requests`. What happens to the user experience in the UI?

---

## Level 11: System Evolution & Modification Scenarios

66. How would you add a 9th agent—a "Regulatory Protocol Compliance Agent"—to the pipeline? What files must be modified?
67. How would you replace the local file storage with AWS S3 without modifying any of the agent classes?
68. If a new study uses French column names (`Date de visite`, `Numéro de sujet`), how do you configure the system to support it without changing Python code?
69. How would you add real-time Slack or email notifications when a `CRITICAL` severity alert is created?
70. How would you convert the synchronous agent loop into parallel worker execution using Python's `concurrent.futures`?

---

## Level 12: Ownership & Behavioral Questions

71. What was your exact contribution to this codebase?
72. What was the single most difficult technical challenge you encountered while building this system?
73. What is the biggest architectural mistake or technical debt currently in this codebase, and how would you fix it?
74. How did you verify that your risk scoring algorithm was mathematically sound and clinically reasonable?
75. If you had an additional two weeks to work on this platform, what would you prioritize building?

---

## Level 13: Tough Interviewer Grill Questions

76. *"Why do you call this 'Agentic AI'? Isn't this just a traditional data pipeline with a few if-statements and an LLM prompt tacked onto the end?"*
77. *"Why did you write your own EventBus instead of using a battle-tested library like PyDispatcher or Celery?"*
78. *"Your risk scoring weights are arbitrary (0.50, 0.25, 0.25). How can you defend these weights to an FDA auditor?"*
79. *"Why does the repo have both Flask templates and Streamlit? Is this project a messy mashup of two unfinished prototypes?"*
80. *"If an Excel sheet has 50 columns, your schema agent only standardizes 15 known aliases. What happens to the other 35 columns? Are they dropped, or do they leak unstandardized data downstream?"*
81. *"Your quality score gives equal 50% weight to completeness, but in clinical trials, a missing visit date is far more dangerous than a missing middle name. Isn't your quality formula fatally flawed?"*
82. *"Why are you writing Parquet files to local disk? If this app runs in a Docker container, won't all data vanish as soon as the container restarts?"*
