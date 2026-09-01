# ContextCart Local Frontend Demo

The `demo/app.py` file serves as a lightweight presentation layer for the existing `solution.agent.Agent`. The frontend utilizes single-page HTML/CSS/JavaScript within the repository, while the backend relies solely on Python's standard library HTTP server. Consequently, there is no need for Gradio, Node.js, cloud services, or API keys. It does not modify the product catalog, retrieval logic, or official evaluators; all recommended ASINs are drawn from a read-only 50K competition catalog.

## Installation and Startup

Execute the following commands in the repository root directory:

```powershell
python -m pip install -r requirements.txt
python demo/app.py
```

Open your browser at <http://127.0.0.1:7860>. The service listens only on localhost by default and does not create a public link. The first startup requires loading the 50K catalog and memory index; wait for the overlay to disappear automatically and display "System Ready" before beginning the demo.

If the port is in use:

```powershell
python demo/app.py --port 7861
```

## Recommended Live Workflow

Send the following prompts sequentially:

1. `Show me some ideas.`
2. `For that, what matters is: waterproof.`
3. `Actually, ignore my earlier preference. What I need is: material: leather; category: shoes; under $100.`

After the first round of prompts is sent, the input box and send button become available again. A single page can complete up to 10 consecutive rounds. The page synchronously displays:

- Buying/Browsing route and intent confidence;
- Over-Generality truncation and active clarification;
- Number of multi-path candidates, runtime Context Program, and current round constraints;
- Top-10 products, ASINs, categories, prices, and ratings;
- Distilled context, Intent Override, whether the LLM was applied, and reasons for safety degradation.

The "Verify Evidence" page only reads the saved dev150 aggregated summaries from the repository and does not read evaluation labels, target ASINs, or re-run public/holdout during the demo.

## Local API

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/status` | Initialize, catalog, Dense, and LLM status |
| GET | `/api/scenarios` | Fixed demo scenarios without labels |
| GET | `/api/evidence` | Saved dev150 aggregated evidence |
| POST | `/api/reset` | Create an isolated session |
| POST | `/api/respond` | Execute a round of real Agent reasoning |
| POST | `/api/llm` | Explicitly enable or disable local Qwen sorter |

The request body is limited to 64 KB, with a maximum of 2,000 characters per message. Sessions are strictly limited to 10 rounds. Agent initialization and invocation are fixed to the same dedicated thread to avoid SQLite FTS connection cross-thread issues causing hangs on the second round.

## Optional Qwen

By default, Qwen remains disabled to utilize the verified deterministic scheme. If Ollama is already running on the live machine, you can explicitly enable it from the page or configure it before startup:

```powershell
$env:CONTEXTCART_LLM_ENABLED = "true"
$env:CONTEXTCART_LLM_MODEL = "qwen3.5:9b"
python demo/app.py
```

Other environment variables include `CONTEXTCART_WEB_PORT`, `CONTEXTCART_LLM_BACKEND`, `CONTEXTCART_LLM_BASE_URL`, `CONTEXTCART_LLM_TIMEOUT_MS`, and `CONTEXTCART_LLM_TOP_N`. Even if Ollama is unavailable, strict validation and deterministic fallback will not cause the entire Agent to fail.

## Design Boundaries

- The page fixes the Top-10 list and enforces the competition's 10-round limit.
- Verified configurations are used by default; Qwen does not automatically enable upon page startup.
- Multiple browser sessions are isolated with random `session_id`; the catalog and index are loaded only once.
- The page does not touch evaluation targets, does not participate in hyperparameter tuning, and does not alter any evaluation results.
- Stopping the terminal process (e.g., `Ctrl+C`) stops the service.

Terminal fallback demo:

```powershell
python demo/cli_demo.py
```
