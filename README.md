# DataPilot: Automating the Repetitive, Amplifying the Analytical

## Project Background

Data science workflows are riddled with repetitions like dataset profiling, quality checks, baseline modeling, artifact logging. These tasks are necessary but rarely where an analyst's time is best spent. Data-Pilot was built to absorb that overhead. **Can a terminal-first AI copilot handle the mechanical groundwork of data analysis so analysts can focus on what actually requires judgment?**

DataPilot is an interactive, terminal-first data-science copilot powered by the `openai-agents` runtime. It wraps an opinionated analysis agent **Vanessa** with a rich CLI, a curated toolbelt for filesystem, dataset, and automation tasks, and a sandboxed workspace under `./root` for reproducible experiments.

![image](https://github.com/anikaa20/DataPilot--AI-Agent/blob/main/data_pilot_architecture.png)

The system is organized around four capability pillars:

- **Agent Core**: Vanessa's planning loop, dataset-first guardrails, and turn management
- **Toolbelt**: filesystem operations, dataset profiling, code execution, and automated modeling
- **CLI Experience**: Rich-powered streaming interface with slash commands and session control
- **Extensibility**: plug-in architecture for new tools, agents, and handoff configurations

Full project repository → [`Data-Pilot`](https://github.com/anikaa20/DataPilot--AI-Agent)

---

## Architecture Overview

**Source:** Single-agent system with a modular toolbelt, `openai-agents` runtime + `LitellmModel` bridge

**Repository structure:** 9 core modules across `cli/`, `config/`, `my_agents/`, `tools/`, and `root/`

```
Data-Pilot/
├── main.py                  # Async entrypoint launches the CLI and agent loop
├── cli/
│   └── ui.py                # Rich UI, slash commands, streaming visualizer
├── config/
│   └── agent_config.py      # Version, turn limits, environment-based config
├── my_agents/
│   ├── base_agent.py        # my_agent wrapper around openai-agents runtime
│   └── analysis_agent/      # Vanessa's prompt, tools, and handoff metadata
├── tools/
│   ├── data_tools.py        # Dataset overview, quality, and correlation reports
│   ├── automation_tools.py  # End-to-end modeling pipeline + artifact logging
│   ├── filesystem_tools.py  # Sandboxed file ops within ./root
│   ├── misc_tools.py        # execute_code + timestamps
│   └── utils/               # Sandbox, dataset, and code-execution helpers
├── root/                    # User-editable sandbox for datasets and outputs
└── README.md
```

| Control Flow Step | What Happens |
|---|---|
| Launch | `main.py` runs `cli.ui.run_cli`, greeting the user and opening the interactive loop |
| Delegation | CLI delegates user turns to `analysis_agent.agent`, a `my_agent` instance with Vanessa's prompt and full toolbelt |
| Execution | CLI streams reasoning tokens, tool calls, and handoffs while enforcing `MAX_TURNS` (default: 20) |
| Sandboxing | All artifacts and datasets are manipulated inside `./root`; Repository code is never touched |

---

## Why DataPilot

Data-Pilot is not a generic chatbot wrapper. Every design decision reinforces a single principle: **the agent should do the mechanical work, not just describe it.**

* Vanessa insists on a dataset path under `./root` before taking any action as provenance and reproducibility are non-negotiable, not optional.
* The `execute_code` tool runs actual Python in a sandboxed environment, logging XML-formatted stdout/stderr for full transparency on every step.
* `automated_modeling_workflow` delivers a complete baseline modeling pass such as preprocessing, training, evaluation, and artifact logging in a single tool call, without the analyst writing a line of boilerplate.
* The Rich-powered CLI streams reasoning and tool calls in real time, so the analyst sees *what* Vanessa is doing and *why* at every turn and not just a final answer.
* The `my_agent` wrapper makes the entire stack extensible: new tools, new agents, and multi-agent handoffs require minimal configuration changes.

---

## Capabilities Deep Dive

### Capability 1: Agent Core & Dataset Guardrails
* Vanessa opens every session by collecting five inputs: dataset path, business objective, target variable, success criteria, and constraints. No action is taken until this context is established.
* `MAX_TURNS` (default: 20) enforces a hard ceiling on agentic loops, preventing runaway execution without analyst oversight.
* Supported dataset formats span CSV, TSV, JSON/NDJSON, Parquet, and Excel. The agent auto-detects format and surfaces schema details before committing to any analysis plan.

### Capability 2: Toolbelt: From Filesystem to Modeling
* **Filesystem tools** (`list_files`, `read_file`, `write_file`, `edit_file_section`, and more) are fully sandboxed to `./root`. The agent cannot escape the workspace boundary by design.
* **Data tools** (`dataset_overview`, `dataset_quality_report`, `dataset_correlation_report`) deliver instant EDA snapshots: schema, missingness, cardinality, numeric/categorical stats, and correlations without the analyst writing profiling code.
* **Automation tools** (`automated_modeling_workflow`) run end-to-end baseline pipelines: feature preprocessing, model training, metric logging (accuracy, F1, ROC-AUC for classification; R²/MAE/RMSE for regression), and artifact export to `root/analysis_outputs/<session>/`.
* **Misc tools** (`execute_code`, `get_current_datetime`) handle arbitrary Python execution with timeout enforcement and timestamped session tracking.

### Capability 3: CLI Experience
* The Rich-powered interface streams reasoning, tool invocations, and responses in real time enabling analysts to see the agent's work as it unfolds, not after the fact.
* Slash commands give session-level control without breaking the conversational flow:

| Command | Aliases | Description |
|---|---|---|
| `/help` | `/h` | Display available commands |
| `/history` | `/hs` | Show current conversation transcript |
| `/clear` | `/c` | Clear the screen |
| `/clear_history` | `/ch` | Erase stored conversation memory |
| `/quit` | `/exit`, `/q` | Exit the program gracefully |

* **Ctrl+X** interrupts a streaming response mid-execution. This is useful when Vanessa's plan needs course correction before it completes.

### Capability 4: Extensibility & Multi-Agent Architecture
* New tools require a single decorated Python function appended to any tool list. `my_agent` automatically merges them into the agent at instantiation.
* New agents follow a lightweight convention: define a prompt and config under `my_agents/<name>/`, register in `agent_config.py`, and instantiate via `my_agent`.
* Handoffs are first-class: `analysis_agent.add_handoffs(other_agent)` enables multi-agent collaboration while preserving Vanessa as the orchestrator thus no rewiring of the base agent required.

---

## Setup & Installation

```powershell
# Create and activate a virtual environment
python -m venv .venv
.\.venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt   # or: uv sync

# Configure environment variables
# Copy .env.example → .env and populate:
# CEREBRAS_BASE_URL   e.g. https://api.cerebras.ai/v1
# ANALYSIS_API_KEY    your API token for the selected model

# Launch
python main.py
```

* **Python ≥ 3.10** required. Compatible with any OpenAI-SDK-compatible endpoint, cerebras is the default but not the only option.
* Place datasets under `root/` and reference them with relative paths (e.g., `data/loans.csv`). Anything outside `./root` is inaccessible to the agent by design.
* Generated outputs such as cleaned data, charts, trained models are written to `root/analysis_outputs/<session>/` automatically.

---

## Assumptions & Caveats

* **Sandbox boundary is enforced, not foolproof.** `./root` isolation prevents accidental repository edits during normal use, but it is a path-guard convention, not a containerized security boundary. Do not expose Data-Pilot to untrusted dataset inputs in production environments without additional hardening.
* **`MAX_TURNS = 20` is a safeguard, not a ceiling for complex tasks.** Long analyses may require the analyst to continue across multiple sessions; single-session completeness is not guaranteed for large or deeply nested workflows.
* **Baseline models are starting points, not final answers.** `automated_modeling_workflow` produces benchmark metrics to orient the analysis  such as -> hyperparameter tuning, feature engineering, and model selection remain the analyst's responsibility.
* **API dependency introduces latency and quota risk.** Vanessa's planning and execution loops consume API tokens per turn; high-`MAX_TURNS` sessions on large datasets can exhaust quota faster than expected. Monitor usage during extended runs.
* **Cerebras is the default endpoint but not a hard requirement.** Any OpenAI-SDK-compatible API can be substituted via `.env`.Model behavior and performance will vary depending on the underlying provider.
