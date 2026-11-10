# CLAUDE.md: IncidentIQ AgentOps build guide

This file tells Claude (Claude Code) how to build this project. Read **README.md** first: it is the design spec. **INFO.md** is the interview prep companion. This file is the build plan.

## Goal

When an alert fires, a LangGraph multi-agent workflow does the on-call engineer's first pass on IronPay (`ironpay-ledgerapi`):

- a triage agent reads the Alertmanager alert and picks which specialists to run;
- four specialist agents pull evidence in parallel through MCP tool servers: logs from Loki, metrics from Prometheus, recent deploys and config from git, and runbooks through RAG over markdown runbooks in ChromaDB;
- a reasoning agent ranks the top three likely root causes, each citing evidence IDs, and proposes a fix;
- a human-in-the-loop approval step (LangGraph `interrupt`) comes before any action;
- a summary agent writes the post-incident summary.

IronPay is broken on purpose in 15–20 scripted ways (crashed container, slow query, bad config, memory pressure, bad deploy). An eval harness scores IncidentIQ per incident: root cause in top-1 and top-3, time to first hypothesis, tokens and USD per incident. A small web UI shows the incident timeline and handles approvals.

This is a 2–4 week, one-person portfolio project. Keep it at that size.

## Tech stack

- Python 3.11 (pip + pinned `requirements.txt` and `requirements-dev.txt`)
- LangGraph (StateGraph, `Send` fan-out, `SqliteSaver` checkpointer, `interrupt` / `Command(resume=...)`)
- MCP Python SDK (FastMCP servers over streamable HTTP; MCP client in the agents)
- Claude through the official `anthropic` SDK: `claude-haiku-5-5` for triage, specialists and summary; `claude-opus-5-5` for root-cause reasoning
- ChromaDB (persistent, embedded in the runbooks MCP server, default local embedding model)
- Prometheus, Alertmanager, cAdvisor, Postgres and Redis exporters
- Loki, Grafana Alloy, Grafana
- FastAPI (alert webhook, incident API, approval resume) and SQLite (incidents, checkpoints, cost log)
- Web UI: Next.js + Tailwind CSS + lucide-react
- Docker Compose (includes IronPay's compose file from a local clone set by `IRONPAY_DIR`), Makefile
- pytest, ruff, GitHub Actions
- Supporting libraries: pydantic, pydantic-settings, python-dotenv, httpx, GitPython

## Build order (phases)

1. **Scaffold and observability stack.** Repo layout from README.md, `docker-compose.yml` that includes IronPay and adds Prometheus (with alert rules for 5xx rate, p95 latency, target down, container restarts, consumer lag), Alertmanager (webhook to IncidentIQ), Loki, Grafana Alloy, Grafana, cAdvisor and exporters. Makefile (`up`, `down`, `test`). `.env.example`. CI workflow (ruff + pytest) green from day one.
2. **LLM module `incidentiq/llm/`.** Thin wrapper on the `anthropic` SDK: model per node, effort set explicitly per node, `stop_reason` checks, structured outputs helper, token and USD log per call (including cache tokens) from a price table in config, per-incident cap (`INCIDENTIQ_MAX_USD_PER_INCIDENT`) and monthly cap (`LLM_MONTHLY_BUDGET_USD`), fake provider for tests.
3. **MCP servers for logs, metrics and changes.** Read-only tools that summarize server-side (grouped error patterns with counts, downsampled series with baseline, commits and config diffs in a window). Tests use recorded Loki and Prometheus responses.
4. **Runbook RAG.** Write 12–15 short markdown runbooks for IronPay, chunk by heading with metadata, load into ChromaDB, runbooks MCP server with `search_runbooks` and `get_runbook`. Small retrieval test set (symptom query to expected runbook).
5. **LangGraph workflow.** Typed state with reducers; triage node; specialists fanned out with `Send`, each a tool-use loop with a 6-call limit; reasoner with structured output (up to 3 hypotheses with `cause_id`, confidence, evidence IDs, proposed fix); citation validation with one retry; at most one loop back for more evidence; approval node with `interrupt`; summary node; SQLite checkpointer. Graph tests with the fake provider, including interrupt and resume.
6. **API and web UI.** FastAPI: `POST /alerts` (202, dedupe by fingerprint, run in background), incident list and detail, `POST /incidents/{id}/decision` that resumes the graph. Next.js UI: incident list, incident timeline with evidence and the approval panel, eval results table. Run `/unslop` on the main screens, then capture real screenshots into `docs/screenshots/`.
7. **Incident catalog.** `incidents/catalog.yaml` with 15–20 scenarios across crashed container, slow query, bad config, memory pressure, dependency down and bad deploy. Each has inject and reset scripts, the expected alert and a ground-truth `cause_id` from the taxonomy (include taxonomy entries that no scenario uses, plus `other`).
8. **Eval harness and results.** Live mode (reset, inject, wait for alert, investigate, auto-approve, score) that saves evidence snapshots; replay mode that re-runs the reasoning step from snapshots. Metrics: top-1 and top-3 accuracy, median time to first hypothesis, tokens and USD per incident, share of steps flagged as needing human correction. Raw output to `docs/results/`, results table in README.md, demo GIF.

## Stretch goals

Only after all eight phases are done and the README results are filled from real runs:

- Slack or Teams notifications with an approve button that resumes the graph
- Auto-remediation for safe actions (container restart, rollback to the previous image) after approval, limited to an allowlist of commands
- OpenAI as an alternate provider through the official `openai` SDK, compared on the same incident set
- Tracing with Langfuse
- Replay evals of the reasoning step through the Message Batches API

## Working rules for Claude

- Keep the scope at the level in README.md: this is a portfolio project for a master's graduate with about a year of experience. Do not add components the README does not list. The owner must be able to explain every line in an interview.
- Build **one phase at a time**, in order. Each phase must run end to end before the next starts.
- After each phase, update the **Roadmap checklist in README.md** and commit with a clear message.
- **Never invent results.** Every number in README.md and INFO.md (accuracy, latency, throughput, cost) must come from a run in this repo. Save the raw output in `docs/results/`.
- **Screenshots:** when a phase produces something visible (UI, Grafana dashboard, test or eval report), capture real screenshots into `docs/screenshots/` and embed them in README.md. Never mock them up.
- The UI follows the unslop skill (`~/.claude/skills/unslop`): run `/unslop` on the main screens before calling the UI done.
- Keep secrets out of git: `.env` (gitignored) plus a committed `.env.example`.
- Never commit datasets or large model files. Write download scripts and gitignore the data folders.
- Python 3.11+. Dependencies live in a pinned `requirements.txt` (runtime) and `requirements-dev.txt` (pytest, ruff). Setup is `pip install -r requirements.txt`. Use `ruff` for lint/format, type hints, and `pytest` for tests.
- All MCP tools are read-only in the core scope. Nothing in the core scope executes a fix; approval records the decision and shows the command for a human to run.
- **LLM rules:**
  - Every LLM call goes through the `incidentiq/llm/` module, which calls Claude through the official `anthropic` SDK. (OpenAI through the `openai` SDK is a stretch goal; keep the module's interface small so it can be added later.)
  - Keys come from `.env` with a committed `.env.example`. Never log keys.
  - Defaults are `claude-opus-5-5` for the reasoning node and `claude-haiku-5-5` for triage, specialists and summary. Both are configurable in `.env`.
  - Set effort explicitly per node (Opus 5.5 defaults to `medium` and its thinking cannot be disabled). On Opus 5.5, forced `tool_choice` (`any` or a named tool) returns a 400: use structured outputs (`client.messages.parse()` with Pydantic) for JSON.
  - Check `stop_reason` (including `refusal` and `max_tokens`) before reading a response.
  - Put stable prompts and sorted tool definitions first and use prompt caching; verify with `cache_read_input_tokens`.
  - Log tokens and cost for every call, and enforce the per-incident (`INCIDENTIQ_MAX_USD_PER_INCIDENT`) and monthly (`LLM_MONTHLY_BUDGET_USD`) caps. The total budget is about $100: keep the eval set small and use replay mode for prompt iteration.
  - Tests use a fake provider, so CI never spends money or needs keys.
- `make up` (or `docker compose up -d`) must bring up the demo locally.
- Keep the GitHub Actions workflow (ruff, pytest, UI lint and build, `docker compose config`) green. Never run the live eval in CI.
- Prefer small, readable modules. Add a short "How it works" note to `docs/` after each phase.

## Definition of done

- One-command local demo (`make up` or `docker compose up -d`)
- README has the architecture diagram, setup steps, real screenshots, a demo GIF and measured results
- CI is green and tests cover the core logic (graph routing, citation checks, interrupt and resume, MCP tools, scoring)
- The UI has passed an `/unslop` review

## Status

Only `README.md`, `INFO.md` and this file exist so far. Start at **Phase 1**.
