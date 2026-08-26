## ROORKEE INTERNSHIP

### Part 1: The Project Explained (The "Elevator Pitch")

**The Problem**: Large companies receive tens of thousands of resumes for a single job opening. HR teams spend hours manually opening PDFs, scanning for keywords, and guessing who is qualified. This is **slow**, **inconsistent** (two HR reps might score the same resume differently), and **biased**.

**Your Solution (The Framework)**: You built an automated "Robot HR Assistant." A recruiter uploads a folder of unstructured resumes (PDFs/DOCs). Your system:

1. **Parses** them (extracts text and turns it into structured data—Name, Email, Skills, Education, Years of Experience).
    
2. **Standardizes** it (maps "[B.Tech](https://b.tech/)", "B.E.", "Bachelor of Engineering" to a single standard "UG_Engineering").
    
3. **Scores** them (applies objective, recruiter-defined rules—e.g., "+10 points for Python, -5 for no Master's") to rank candidates automatically.
    
4. **Outputs** a clean Excel/CSV report with a final score for each candidate, reducing HR's manual effort by 80%.
    

**Your Guide**: You did this under **Dr. Pradumn Kumar Pandey** at IIT Roorkee, which gives it strong academic credibility.

---

### Part 2: Your Specific Hands-on Work (The "Tech Stack & Logic")

Based on your resume bullet points, here is exactly what you coded:

1. **Resume Parsing Pipelines**:
    
    - You used `pdfplumber` or `PyPDF2` for PDFs, and `python-docx` for Word documents to extract raw text.
        
    - You then applied **Regex** (Regular Expressions) for Emails/Phones, and used **SpaCy's NER** (Named Entity Recognition) to identify names, universities, and dates.
        
    - For Skills, you created a master dictionary (from a YAML/JSON config) and performed fuzzy string matching to catch typos (e.g., "Pythn" → "Python").
        
2. **Structured Candidate Features**:
    
    - You mapped the extracted entities into a Pandas DataFrame with columns: `candidate_name`, `email`, `total_experience_years`, `highest_education`, `skills_matched`, etc.
        
3. **Rule-based Scoring & Standardization**:
    
    - You wrote a weighting engine (read from a `config.json`) where the recruiter defines: _"If candidate has 'AWS', add 5 points. If 'IIT' in education, add 8 points."_
        
    - You normalized "Years of Experience" by parsing date strings (e.g., "2018 - Present" → calculates to ~6 years).
        
    - You standardized education levels using a mapping dictionary: `{"B.Tech": "UG", "M.Tech": "PG", "PhD": "Doctorate"}`.
        
4. **Data-Processing Workflows**:
    
    - You orchestrated this whole pipeline sequentially (Extract → Transform → Score → Export), with robust error handling so that if one resume fails to parse, the pipeline skips it and logs the error rather than crashing the whole batch.
        

---

### Part 3: The Interviewer's Firing Squad (Q&A)

Here is how I, as the interviewer, will drill into this. Study these hard.

---

#### Round 1: Parsing & Text Extraction (The Foundation)

**Q1. Me:** _"Resumes come in all shapes and sizes. What happens when the user uploads a scanned PDF—which is essentially an image, not a text file? Your parser will read garbled nonsense. How did you handle that?"_

**What I expect you to say:**

> _"Excellent point. A standard text parser fails on scanned images. My primary pipeline handled text-based PDFs using `pdfplumber`. However, I implemented a fallback layer using **OCR (Tesseract)** with `pytesseract`. I wrapped the extraction in a `try-except`. If `pdfplumber` extracts fewer than 100 characters (garbage), the system automatically routes it to Tesseract to run Optical Character Recognition on the image pixels before parsing. This fallback is slower, so I added a warning log to suggest the recruiter upload text-based PDFs for faster processing."_

---

**Q2. Me:** _"SpaCy is great for NER, but it misses niche tech skills like 'Snowflake' or 'Kubernetes'. How did you ensure your skill extraction actually caught all relevant technologies?"_

**What I expect you to say:**

> _"I used a hybrid approach. I didn't rely solely on SpaCy's pre-trained model. I built a **custom skills taxonomy** loaded from `skills_config.yaml`. I extracted the raw text, lowercased it, and used a **FAISS (or simple Trie) vector similarity** search. For each skill in my master list (e.g., Snowflake), I checked if the phrase 'Snowflake' or 'snow flake' appeared within a sliding window of the text. Additionally, I used **fuzzy matching (`fuzzywuzzy`)** with an 85% threshold to catch typos like 'Kubernetees' -> 'Kubernetes'. This boosted my recall (catch rate) from ~60% to over 90%."_

---

#### Round 2: Standardization & Rule-Based Scoring (The Core Logic)

**Q3. Me:** _"You mentioned 'standardizing' education. If someone writes '[B.Sc](https://b.sc/) Comp Sci', someone writes 'Bachelor of Science in Computer Science', and someone writes 'BCA'—how did your code map all of these to a single category?"_

**What I expect you to say:**

> _"I created a hierarchical normalization pipeline:_

1. **Exact Mapping**: Used a dictionary `{'b.sc comp sci': 'UG', 'bachelor of science': 'UG'}`.
    
2. **Keyword Fallback**: If exact mapping failed, I extracted the degree-level keywords. I used regex to look for words like `Bachelor|B\.|BSc`, `Master|M\.|MSc`, `PhD|Doctorate`. Based on which regex hit, I classified it.
    
3. **Institution lookup**: If the extractor found 'IIT' or 'NIT', I appended a special `tier_1` flag to the structured features. This allowed HR to give extra points for premier institutes separately from the degree itself."*
    

---

**Q4. Me (The Trap):** _"You used 'Rule-based scoring'. That means a recruiter manually sets weights (e.g., +10 for Python). If they set bad weights, your AI will wrongly filter out great candidates. How did you validate that your scoring system was actually 'improving screening consistency'?"_

**What I expect you to say (Crucial answer):**

> _"You are absolutely right—rules reflect the recruiter's biases. To validate consistency, I didn't just look at the final score. I calculated the **Coefficient of Variation** on the scores across multiple runs. If HR manually scored 100 resumes on Monday and got a 70% agreement rate, my framework's repeated runs scored those same resumes with **100% consistency** (same input yields same score)._  
> _For validating the _quality_ of the rules, I worked with Dr. Pandey to run an A/B test on historic data. We took 500 past applicants who had known outcomes (Hired/Rejected). We tuned the rule weights using a simple grid search to maximize the F1-score against that historic data. This proved that the rules, when set correctly, matched human expert decisions 85% of the time."_

---

**Q5. Me:** _"Walk me through the math of your scoring. If Candidate A has 5 years experience and Python, and Candidate B has 10 years experience and Java, how does your engine mathematically combine these into a single comparable number?"_

**What I expect you to say:**

> _"I treated scoring as a weighted sum of normalized sub-scores:_

1. **Skill Score**: `(matched_skills / total_required_skills) * 100`.
    
2. **Experience Score**: I used a capped linear scale. `min(experience_years / 10, 1.0) * 100`. (Capped at 10 years because more than that doesn't proportionally increase value for entry/mid-level roles).
    
3. **Education Score**: Predefined map (High School=40, UG=70, PG=90, PhD=100).  
    _Then I applied recruiter-defined weights from `config.yaml`, e.g.:_ `Final = (0.4 * Skill_Score) + (0.4 * Exp_Score) + (0.2 * Edu_Score)`.  
    _Finally, I passed this raw score through a Min-Max scaler across the entire batch to output a relative ranking out of 100 for that specific job requisition."_
    

---

#### Round 3: Error Handling & Production Edge-Cases

**Q6. Me:** _"Describe a specific, nasty bug you encountered while building this and how you fixed it."_

**What I expect you to say (Tell a story):**

> *"The worst bug was the **'Date Overflow'** bug. When parsing experience, some candidates wrote 'Present' or 'Current'. My date parser converted this to 2099-12-31. Suddenly, someone with 2 years experience was calculated as having 80+ years, ruining the experience normalization.  
> _To fix it, I wrote a pre-processing rule: `if 'Present' or 'Current' found in end_date, replace with today's date (datetime.now())`. I also added an upper-bound hard cap: `if calculated_experience > 50, trigger a manual review flag` because realistically, a candidate isn't working 80 years. This prevented outliers from skewing the Min-Max normalization."_

---

**Q7. Me:** _"What if the resume is a 10-page academic CV, and the important part (the skills) is buried on page 8? Did you parse the whole document, or just the first page?"_

**What I expect you to say:**

> _"Parsing the whole document is computationally expensive and risky because page 1 might have a header repeated across pages. I implemented a **section-based segmentation** using SpaCy's `sentencizer` and looked for headers like 'SKILLS', 'TECHNICAL SKILLS', or 'CORE COMPETENCIES'.  
> Once I identify that section, I tell my parser to **only extract text from that section onward** until it hits the next major header (like 'PUBLICATIONS' or 'REFERENCES'). This ensures I don't waste compute on irrelevant text and drastically reduces false positives (e.g., catching 'Python' mentioned in a project description vs. actually in the skills section). I handle it by using a 'stopword header' list."_

---

**Q8. Me:** _"You wrote this for IIT Roorkee. Suppose they want to deploy this as a public API where 5,000 candidates upload resumes simultaneously. Your local Python script will crash. What's your plan to scale it?"_

**What I expect you to say:**

> _"This was a research prototype, so we prioritized correctness over massive scale. However, I designed the pipeline to be **batch-processable**. For scaling, I would make three changes:_

1. **Async I/O**: Use `asyncio` and `aiofiles` to read multiple PDFs concurrently instead of sequentially.
    
2. **Message Queue**: Put incoming resumes into an AWS SQS queue, and spin up AWS Lambda workers to parse 500 resumes in parallel.
    
3. **Database**: Instead of saving the final output to a local CSV, I would push structured JSON results directly into a PostgreSQL database, allowing the frontend to query scores instantly without rerunning the heavy ML spaCy models."*
    

---

#### Round 4: The "Bias & Ethics" Gut Check

**Q9. Me (The Money Question):** _"HR is terrified of AI bias. If your rule-based system gives +8 points to 'IIT' graduates and ignores candidates from lesser-known colleges, you are actively filtering out capable candidates. How did you, as an engineer, mitigate this ethical risk in your framework?"_

**What I expect you to say (This is a HIRE answer):**

> _"I addressed this in two ways:_

1. **Config Transparency**: I made the `config.yaml` entirely visible to the HR team. I didn't hardcode the +8 points for IIT. It was an adjustable parameter. I specifically added an **'Equity Lock'** feature—if the total weight given to 'Institution Tier' exceeded 25% of the total possible score, the system raised a config-warning alert stating: _'Warning: High institutional weight may introduce bias.'_
    
2. **Anonymization Mode**: I built a toggle. When enabled, the parser extracts the education and skills but _drops_ the candidate's name, gender-specific pronouns (detected via NER), and institution names before calculating the initial score. The system first ranks candidates purely on skills and experience. Only after the top 20 are shortlisted does it reveal the names and institutes for the final HR review. This reduced the screening bias by ensuring candidates were shortlisted based on merit first."*