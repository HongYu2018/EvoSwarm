# EvoSwarm V1.4: Causal, Adaptive Multi-Agent Intelligence for Supply-Chain Decisions

**Research code and synthetic experimental scenarios** | Python | Reproducibility documentation

EvoSwarm investigates whether a heterogeneous swarm of specialist agents can learn a **minimum sufficient causal capability set** for dynamic supply-chain decisions, rather than always invoking every agent or always minimising swarm size. A temporal–causal world model supplies structured state, historical observations, events and inferred consequences. The joint system can adapt agent activation and communication, request missing expertise during a decision, choose among reasoning levels, update contextual agent trust, and retrieve compact prior experiences.

> **Scope:** This repository is a **synthetic supply-chain simulation**, not a corpus of observed company operations. Results from the deterministic/proxy model do not establish live-LLM performance. Compare systems using operational quality **and** resource consumption; strict binary scenario success alone is insufficient.

## Documentation

- [Dataset generation, scenarios, examples and CSV dictionary](DATASET.md)
- [Installation, experiment protocols, commands and reproduction](EXPERIMENTS.md)
- `EXPERIMENT_DESIGN.md` — accompanying hypotheses/design notes supplied in the code archive.
- `VALIDATION_NOTES.md` — historical V1.4 validation notes; check the protocol and output-schema caveats below before quoting numbers.

## Architecture

```text
Synthetic scenario and timeline (evoswarm/scenarios.py)
                  |
                  v
Supply-chain environment (evoswarm/environment.py)
                  |
                  v
Temporal–causal World Model (evoswarm/world_model.py)
                  |
       +----------+------------------+
       |                             |
Specialist L1 agents (A1–A8)    Cognitive Shell
       |                             |
       +--------- shared evidence ---+
                  |
   Adaptive topology / edge policy / contextual trust
                  |
    Reasoning-level decisions (L1 / local L2 / API L3)
                  |
      Procurement, staffing, fulfilment and dispatch action
                  |
       Simulation outcome + delayed feedback
                  |
       Causal credit / trust / experience updates
```

The modules implement eight specialists: `A1_Demand`, `A2_Supplier`, `A3_Inventory`, `A4_Warehouse`, `A5_Staffing`, `A6_Procurement`, `A7_Fulfilment`, and `A8_Transport`. Execution order is determined by dependencies and is **not** simply A1→A8: transport evidence is obtained before the procurement and fulfilment integrators. See `evoswarm/core.py` and `evoswarm/agents.py`.

The system distinguishes three reasoning levels: lightweight specialist/shell logic (L1), a local Ollama-backed model (L2), and an OpenAI API model (L3). `--llm-mode proxy` uses proxy behaviour rather than making live model calls; `--llm-mode live` enables the configured backends and can incur API charges. Consult `evoswarm/agents.py`/`evoswarm/shell.py` for fallbacks and exact invocation conditions.

## Repository layout

```text
EvoSwarm_Code_Data/
├── evoswarm/
│   ├── agents.py             # specialists, decision assembly and LLM planners
│   ├── config.py             # .env-backed backend settings
│   ├── core.py               # shared agent/state/event data types
│   ├── environment.py        # synthetic supply-chain simulation
│   ├── experiment.py         # system comparisons, metrics and result export
│   ├── learning.py           # adaptive topology, level policy and trust
│   ├── memory.py             # symbolic/causal experience capsules
│   ├── plots.py              # generated result charts
│   ├── scenarios.py          # scenario generator and fixed manifest
│   ├── shell.py              # cognitive shell
│   └── world_model.py        # state, events and causal reasoning
├── run_experiment.py         # single-seed experiment CLI
├── run_multiseed.py          # multi-seed aggregate CLI
├── analyse_results.py       # read/print summaries from output directory
├── check_backends.py        # live Ollama/OpenAI connectivity check
├── requirements.txt
├── .env.example
├── EXPERIMENT_DESIGN.md
├── VALIDATION_NOTES.md
├── QUICK_START.txt
└── outputs_*/              # included historical CSVs and figures
```

A substantial portion of the supplied archive consists of **pre-generated output folders**, not independent external training datasets. Some archived results are from earlier internal revisions and have different column schemas from the current Python code; use output metadata and code version when reporting results.

## Quick start (Windows PowerShell)

From the repository root:

```powershell
py -3 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
.\.venv\Scripts\python.exe run_experiment.py --episodes 10 --steps 6 --seed 7 --llm-mode proxy --output-dir outputs_quickstart
.\.venv\Scripts\python.exe analyse_results.py --output-dir outputs_quickstart
```

Using `\.venv\Scripts\python.exe` does not require PowerShell script execution-policy changes. On Linux/macOS, substitute `.venv/bin/python` for `\.venv\Scripts\python.exe`. No Ollama daemon or OpenAI key is necessary for a **proxy-mode** test.

For full setup, live backend configuration, experiment variants and common errors, see [EXPERIMENTS.md](EXPERIMENTS.md).

## Experiment systems

| CLI identifier | Description |
|---|---|
| `full_l1_shell` | All eight agents, L1/shell only |
| `expert_fixed_swarm` | Human-defined feature-to-specialist rule mapping |
| `fixed_hybrid` | All eight specialists with hand-designed reasoning escalation |
| `vertical_evoswarm` | Full specialist set and learned reasoning-level adaptation |
| `joint_evoswarm_cold` | Adaptive joint architecture reset between episodes; no accumulated cross-episode experience |
| `joint_evoswarm` | Joint topology/communication/level adaptation and cross-episode learning/memory |
| `pure_l2` | Local-model-only planner baseline |
| `pure_l3` | Frontier-model-only planner baseline (no specialist outputs) |

The `--systems` flag selects any subset from this list. For *live* `pure_l3`, also provide `--include-pure-l3`; for proxy experiments see the source's baseline controls and metadata. `run_multiseed.py` does **not** expose `--systems`; it executes the default comparison set.

## Data and evaluation at a glance

The source contains **10 scenario families**: seven base families and three compound generalisation families. Each episode creates a procedurally varied environment with demand, inventory, backlog, supplier risk, delivery/transport, staffing, capacity, operational events and delayed outcomes. The scenario generator draws random quantities from ranges, with the provided seed controlling stochasticity. It also introduces context-dependent specialist biases **hidden from the world-model views** to test trust adaptation.

The **current generator uses a fixed 10-episode manifest block**: four `experience` cases, three `recurrence` cases and three `generalisation` cases per block. Thus 120 episodes yield 48/36/36, **not** 40/40/40. Earlier documents and some archived outputs describe an old equal-thirds arrangement. `scenario_manifest.csv` and `run_metadata.json` from a fresh run are the authoritative record of which cases were used. More detail: [DATASET.md](DATASET.md).

Primary evaluation dimensions: operational score and service; urgent-order fulfilment; final backlog and cost; deadline compliance; active/executed agents and causal coverage; model calls/token/latency/resource proxies; and experienced-vs-cold gains. The strict scenario-success indicator is a secondary measure, and it should not be substituted for continuous quality metrics. The current software additionally exports a continuous `scenario_achievement_score`.

## Fast verification used for these instructions

A local proxy-mode test of the uploaded version succeeded with:

```bash
python run_experiment.py --episodes 2 --steps 2 --llm-mode proxy --systems full_l1_shell joint_evoswarm_cold joint_evoswarm --output-dir outputs_smoke
```

This establishes basic execution and CSV/figure generation **only**; it is not a statistical experiment or verification of the live backends. Pandas may print non-fatal `FutureWarning` messages during aggregation.

## Reporting and reproducibility

Use the exact command, seed list, `run_metadata.json`, `scenario_manifest.csv`, dependency versions, mode (`proxy` vs `live`), actual Ollama/OpenAI model versions, and whether network/API fallback occurred. Report mean ± standard deviation across independent seeds and separate continuous operational performance from model-cost proxies. **Do not interpret the configurable resource-cost weights as measured money, power or hardware FLOPs.**

## Citation and licence

Citation details have not been supplied in the archive. Add the final paper title, authors, venue, year and DOI (or `CITATION.cff`) when confirmed. **No licence file was found in the supplied package**; specify a licence before offering reuse permissions. Do not commit real `.env` credentials or API keys.
