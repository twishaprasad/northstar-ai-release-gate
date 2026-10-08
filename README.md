# Northstar AI Release Gate — Real Agent Evals

A portfolio-grade agent-evaluation project built around a synthetic commerce environment and **real model executions**.

The project asks a practical product question: **what failures should block an agent release, and what has the system earned the right to execute autonomously?**

## What is synthetic vs real

**Synthetic:** Northstar Commerce, order records, customer-support scenarios, policy documents, and deterministic ground truth.

**Real:** when you run the experiment with your API key, the model's intent interpretation, retrieval behavior, tool calls, terminal action attempts, repeated-run variability, latency, and token usage are produced by the live agent.

**Never treated as ground truth:** an LLM-generated label.

## Agent workflow

The live agent has six sandboxed tools:

- `get_order`
- `search_policy`
- `issue_refund`
- `create_replacement`
- `deny_case`
- `escalate_case`

Two architectures can be evaluated with the same model and cases:

1. **Agent-first** — the selected terminal action executes directly in the sandbox.
2. **Hybrid** — the same terminal action passes a deterministic policy/authority gate. A blocked action does not execute; the tool tells the agent to escalate.

This makes it possible to evaluate both **model behavior** and **execution architecture**.

## What gets graded

Each run is graded independently on:

- issue understanding
- authoritative order lookup
- policy retrieval
- retrieval relevance
- policy attribution
- terminal-tool use
- whether grounding/retrieval happened before a terminal action
- proposed action correctness
- first terminal action correctness
- final executed action correctness
- unsafe action attempts
- unsafe executions

Repeated runs add:

- per-scenario reliability
- action distributions
- 95% Wilson lower bounds
- worst-case scenario reliability
- critical unsafe action count
- failure-class distribution
- release recommendation

A model can therefore score well overall and still receive **DON'T SHIP** if a critical unsafe execution remains.

## Dataset

Northstar contains:

- 1,000 synthetic orders
- 300 development scenarios
- 200 evaluation scenarios
- 100 holdout scenarios
- 50 adversarial scenarios

See `DATA_CARD.md` for methodology.

## Run the first real experiment

Create a virtual environment and install dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt
cp .env.example .env
```

Add your credentials locally to `.env`:

```text
OPENAI_API_KEY=...
OPENAI_MODEL=<a current OpenAI API model ID available to your account>
```

Do not commit `.env`.

Inspect the planned experiment without making API calls:

```bash
python3 scripts/generate_live_traces.py \
  --split holdout \
  --limit 6 \
  --trials 3 \
  --dry-run
```

Run the first real batch:

```bash
python3 scripts/generate_live_traces.py \
  --split holdout \
  --limit 6 \
  --trials 3
```

That is 6 scenarios × 3 trials × 2 architectures = **36 live agent runs**.

The experiment is written to `data/live_traces/<timestamp>_<model>/`:

- `manifest.json` — reproducibility metadata and hashes
- `runs.jsonl` — observable model/tool traces and grades
- `summary.json` — reliability and release decision

No private chain-of-thought is stored.

## Create the public replay dataset

Once a real batch exists:

```bash
python3 scripts/export_replay_bundle.py data/live_traces/<experiment-folder>
```

This creates `docs/real_runs.json`.

Open `docs/replay.html` (or deploy `docs/` as a static site) to let visitors inspect the recorded experiment without making new API calls or incurring runtime model cost.

The public replay clearly states that business data is synthetic while agent behavior comes from recorded real model executions.

## Tests

```bash
pytest -q
```

Current suite: **9 tests**.

## Why this is an eval project rather than an agent demo

The agent itself is intentionally small. The project is about the release system around it: holdouts, adversarial tests, repeated trials, trajectory grading, severity-aware safety checks, reproducible manifests, and an explicit release gate.

The goal is not to show that an LLM can call a refund tool. The goal is to show **how a product team determines whether that behavior is reliable enough to ship.**