# ContextCart Live Demo and Video Recording Workflow

## Demo Objectives

Demonstrate four key points in 3–5 minutes:

1. The Agent distinguishes between open browsing and high-intent purchasing;
2. Multi-turn state accumulates, and expired preferences are deleted upon Intent Override;
3. Retrieval strategies dynamically adapt based on context and candidate pools;
4. All recommendation results originate from read-only catalogs, with reproducible metrics and safe fallback mechanisms.

## 15 Minutes Before the Demo

Enter the final submission repository in a fresh terminal:

```powershell
cd "<final_submission_repository_path>"
.\.venv\Scripts\python.exe -m unittest discover -s tests -v
.\.venv\Scripts\python.exe demo\app.py
```

Open <http://127.0.0.1:7860> and wait for the green status bar to display:

```text
System Ready · 50,000 Items in Catalog · Dense: hashing
```

Preparation Checklist:

- Set browser zoom to 90%–100%;
- Clear other tabs and system notifications;
- Do not display local usernames, API keys, Ollama logs, or absolute file paths;
- Do not launch Qwen; the main workflow uses the verified default deterministic Agent;
- Use `experiments/analysis/context_programming_dev_summary.json` as a metrics backup;
- Open a separate terminal and prepare `python demo\cli_demo.py` as a browser failure contingency.

## 3–5 Minute Live Script

### 0:00–0:30 Problem and Solution

Script:

> Traditional search treats each turn as independent keywords, but real users first explore, then add conditions, and may suddenly change goals. ContextCart does more than save chat history; it compiles the current context into an executable retrieval program for each turn.

Point to the dev150 metrics card at the top of the page, but explicitly state that it represents development set results and does not represent private scores for 800 identical items.

### 0:30–1:15 Turn 1: Open Browsing and Active Truncation

Input:

```text
Show me some ideas.
```

Expected Display:

- `browsing / DISCOVERY`;
- `retrieval_cutoff = true`;
- Agent actively asks for features;
- Dense/LLM routes are skipped, yet the page still returns Top-10 from the catalog.

Script:

> This is a broad request. The system first halts expensive routes and then proactively asks a high-information-value question to avoid wasting a turn and model latency on meaningless consumption.

### 1:15–2:00 Turn 2: Information Accumulation and Full Recall

Input:

```text
For that, what matters is: waterproof.
```

Expected Display:

- Preferences enter distilled context;
- Truncation is lifted;
- BM25, Metadata, Dense, and other routes are restored;
- Candidate pool and Top-10 are updated;
- Agent continues to ask about the category that remains undetermined.

Script:

> After the user responds, the state machine acquires a new slot, and the Context Program re-authorizes full recall. The right side of the page simultaneously displays which routes each product originates from.

### 2:00–3:00 Turn 3: Intent Override and Precision Buying Route

Input:

```text
Actually, ignore my earlier preference. What I need is: material: leather; category: shoes; under $100.
```

Expected Display:

- `buying / PRECISION`;
- `override = 1`;
- `material:leather` becomes an applied constraint;
- Top-10 consists primarily of shoes, with recall evidence from multiple routes;
- Agent asks for size or other undetermined purchase attributes.

Script:

> "Actually" is not a regular keyword. The system executes an auditable override operation, deleting old preferences that conflict with the new goal while retaining information that remains valid. The current request always takes precedence over long-term profiles.

### 3:00–3:40 Metrics and Engineering Trade-offs

Display metrics cards: HR@10 0.92, MRR 0.607254, MTTC 4.36, TechnicalScore 0.774976.

Script:

> We have also completed a full development set comparison with Qwen3.5 9B. It improves MRR but does not increase Hit Rate; MTTC is slightly worse, and it is 24.48 times slower, so it is not set as the default. This is an evidence-based deployment gate, not a lack of model access.

### 3:40–4:20 Security and Deployability

Script:

> All ASINs are generated from read-only catalogs. Dense indexes must pass catalog SHA verification; safe fallback occurs if files, dependencies, or models are missing. The default path runs offline, without relying on paid APIs, external vector databases, or keys.

## Video Recording Recommendations

- Visual sequence: Title page (5 seconds) → Local frontend three turns → Metrics → Architecture diagram → Limitations and next steps;
- Recommended video length: 3–5 minutes, 1080p, ensuring tables and diagnostic text are readable;
- Do not play the full 150 evaluation items; show the saved summary with SHA instead;
- Do not display Amazon product images or third-party trademarks; the page should only show text fields from the competition catalog;
- Upload to YouTube and set to Public, then paste the link into Devpost and `docs/DEVPOST_DESCRIPTION.md`.

## Failure Contingencies

### Page Unreachable

```powershell
.\.venv\Scripts\python.exe demo\app.py --port 7861
```

Then open <http://127.0.0.1:7861>.

### Abnormal Browser Interaction

Run the terminal demo for the same three turns:

```powershell
.\.venv\Scripts\python.exe demo\cli_demo.py
```

This script outputs routes, candidate counts, truncation status, asked attributes, distilled context, orchestration reasons, and Top-10 results.

### Dense Not Enabled

Check the page status bar. If the SHA-matching hashing index does not exist, execute:

```powershell
.\.venv\Scripts\python.exe scripts\build_embeddings.py --backend hashing
```

Do not download BGE or launch Qwen on-site temporarily.

## Pre-Closing Checklist

- Input box remains usable after three turns;
- Page shows no `Run Failed` message;
- Third turn displays `PRECISION` and `material:leather`;
- Top-10 ASINs have no duplicates;
- Git working directory is clean;
- GitHub and YouTube links in Devpost are publicly accessible;
- Team member contribution text has been updated to reflect actual circumstances.
