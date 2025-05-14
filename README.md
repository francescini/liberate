# liberate
AI Agent Threat and Safeguard assessment

# AI Risk Analyzer – Phase 1

## 1  Project goal
Build a flexible, **LLM‑powered agent** that decomposes a user‑supplied agentic use‑case into its constituent tasks, interfaces, data flows and privileges so that later phases can analyse threats, risks and safeguards.

Phase 1 outputs a **single structured report** (Markdown and/or JSON) that captures the decomposition. Future phases will reuse the same architecture to add risk scoring and mitigation advice.

---

## 2  High‑level architecture
```
                       +--------------+
                       |  User Input  |
                       +------+-------+
                              |
                              v
+---------------+    plan    +--------------+    calls    +-----------------+
|  Orchestrator |----------->|   Planner    |-----------> |  Library Tools  |
+-------+-------+            +-------+------+            +----+------------+
        ^                             |                       |
        | aggregated results          | returns plan          |
        |                             v                       |
        |                    +---------------+                |
        +--------------------|   Reporter    |<---------------+
                              +-------+-------+
                                      |
                                      v
                         outputs/analysis_YYYYMMDD.md
```
* **Orchestrator** – single entry point; maintains conversation context and short‑term memory.
* **Planner** – optional helper LLM that chooses which library tools to invoke and in what order (Model Context Protocol‑compliant plan object).
* **Library tools** – individual Python functions (thin wrappers around prompts or external APIs) that do one job and declare their signature + JSON schema.
* **Reporter** – collates every tool response into a coherent Markdown/JSON artefact.

The design mirrors modern agent frameworks (e.g. OpenAI Functions, LangChain Runnable) while remaining framework‑agnostic.

---

## 3  Folder structure
```
ai-risk-analyzer/
├── README.md                ← this file
├── .env.example             ← template for secrets
├── requirements.txt         ← Python deps (see § 5)
├── src/
│   ├── __init__.py
│   ├── cli.py               ← command‑line entry point
│   ├── orchestrator/
│   │   ├── __init__.py
│   │   ├── orchestrator.py  ← Orchestrator class
│   │   └── planner.py       ← optional planning helper
│   ├── library/
│   │   ├── __init__.py
│   │   └── task_breakdown.py← first tool implementation
│   ├── utils/
│   │   ├── __init__.py
│   │   ├── mcp.py           ← Model Context Protocol helpers
│   │   └── logger.py        ← opinionated logging wrapper
│   └── reporter/
│       ├── __init__.py
│       └── reporter.py      ← assembles final report
├── tests/                   ← pytest unit & integration tests
├── data/
│   └── samples/
│       └── sample_use_case.txt
├── outputs/                 ← generated reports land here
├── .vscode/
│   └── settings.json        ← linting & debugger prefs
└── .gitignore               ← keeps secrets & artefacts out of VCS
```

> **Tip for VS Code** – open the workspace at project root, ensure the Python interpreter points to `.venv` (see next section), and enable “Run in File Dir” for the debugger.

---

## 4  Local setup (macOS / Linux / WSL / Windows)
1. **Clone & enter**
   ```bash
   git clone https://github.com/your‑org/ai‑risk‑analyzer.git
   cd ai‑risk‑analyzer
   ```
2. **Create a virtual env** (Python ⩾ 3.11)
   ```bash
   python -m venv .venv
   source .venv/bin/activate  # „.venv\Scripts\activate“ on Windows
   ```
3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```
4. **Configure secrets**
   ```bash
   cp .env.example .env  # then edit .env
   ```
5. **Run the sample pipeline**
   ```bash
   python -m src.cli --input data/samples/sample_use_case.txt
   # ➜ outputs/analysis_20250514_1530.md
   ```

---

## 5  Configuration (.env & requirements)
`.env.example` shows **all** required variables:
```
OPENAI_API_KEY=
ORCHESTRATOR_MODEL=gpt-4o-mini
MAX_TOKENS=4096
TEMPERATURE=0.2
LOG_LEVEL=INFO
```
*Never commit `.env` – the `.gitignore` already covers it.*

`requirements.txt` (extract):
```
openai>=1.14
python‑dotenv>=1.0
pydantic>=2.7
rich>=13.7
```
Add LangChain, LlamaIndex, or Haystack later if desired.

---

## 6  Execution flow (step‑by‑step)
1. **Input ingestion** – `cli.py` reads the file or STDIN and passes the text to `Orchestrator.analyse()`.
2. **Initial reflection** – the orchestrator embeds the user use‑case into a *system prompt* explaining the overall objective.
3. **Planning (optional)** – if `planner.py` is enabled, the orchestrator asks it to output a JSON list of `{"tool_name": str, "arguments": {...}}` steps (conforming to MCP’s *Plan* schema).
4. **Tool invocation** – for each planned (or self‑selected) step the orchestrator imports the corresponding function from `src.library.*`, passes the arguments, and stores the result.
5. **Aggregation** – the `Reporter` merges raw tool responses into a Markdown document (plus a machine‑readable JSON alongside).
6. **Persistence** – the artefact is timestamped and written to `outputs/`.
7. **Return** – CLI prints the path; API/GUI would stream or upload the same content.

---

## 7  Model Context Protocol (MCP) usage
All prompts, tool manifests and conversation turns are serialised as MCP objects under `src/utils/mcp.py`. This guarantees reproducibility, makes chain‑of‑thought auditable, and allows later phases to attach threat/risk metadata to any step.

---

## 8  Extensibility roadmap
| Phase | Feature                          | Where it plugs in |
|-------|----------------------------------|-------------------|
| 2     | Threat & risk taxonomy           | `library/` risk tools + `Reporter` section |
| 3     | Safeguard recommendation engine  | dedicated tool functions |
| 4     | Web/API front‑end                | FastAPI app in `src/api/` |

Because each tool is just a standalone Python module with a clear signature, adding new analyses is a *drop‑in* operation.

---

## 9  Security & ops best practices
* **Secrets isolation** – only `.env` file + OS keyring, never hard‑code keys.
* **Reproducible environments** – pin dependencies; use `pip‑tools` or `poetry lock` later.
* **Sandbox** – all network calls are whitelisted; external‑tool invocations run in a subprocess with a restricted UID.
* **Logging** – structured logs via `logger.py`; omit PII by default.

---

## 10  Contributing
1. Fork → create feature branch.
2. Write/upgrade unit tests in `tests/`.
3. Run `ruff` & `pytest` – ensure green.
4. Submit PR.

---

© 2025 Wiser Human – MIT Licence

