# Reproducing EvoSwarm V1.4 experiments

These commands were checked against the uploaded source (not copied blindly from older documentation). The `run_experiment.py` CLI accepts `--episodes`, `--steps`, `--seed`, `--llm-mode {proxy,live}`, `--output-dir`, `--include-pure-l3`, and `--systems`. The multi-seed CLI accepts `--seeds` but **does not accept `--systems`**.

## 1. Software requirements

- Python 3 with `venv` and pip; a modern Python 3 release compatible with the listed libraries is recommended. The supplied code archive does not pin an exact interpreter build.
- Required packages from the included `requirements.txt`: `numpy>=1.26`, `pandas>=2.2`, `matplotlib>=3.8`, `python-dotenv>=1.0`, `openai>=2.0`.
- For `live` mode: installed/running Ollama with a local model matching `.env` **and/or** OpenAI API access as required by the systems being executed. Internet access and a paid API account may be needed for L3.
- CPU execution is sufficient for the proxy simulation; the program does not require CUDA for its supplied Python simulator. Live Ollama memory/performance requirements depend on the chosen local model and machine.

### Windows PowerShell (no script activation required)

```powershell
cd path\to\EvoSwarm_Code_Data
py -3 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
```

Using the virtual environment's Python directly avoids PowerShell's unsigned `Activate.ps1` execution-policy problem. If you prefer activation and your policy permits it, run `.\.venv\Scripts\Activate.ps1` and then use `python` instead.

### Linux / macOS

```bash
cd /path/to/EvoSwarm_Code_Data
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -r requirements.txt
cp .env.example .env
```

The remaining examples use `python` for clarity. On Windows replace `python` with `.\.venv\Scripts\python.exe` if you did not activate; on macOS/Linux use `.venv/bin/python`.

## 2. Backends and configuration

The project reads `.env` through `python-dotenv` in `evoswarm/config.py`:

```dotenv
OLLAMA_URL=http://localhost:11434/api/generate
OLLAMA_MODEL=qwen2.5:7b
OPENAI_API_KEY=replace_with_your_actual_key
OPENAI_L3_MODEL=gpt-4.1-mini-2025-04-14
PURE_L3_MODEL=gpt-4.1-mini-2025-04-14
```

Use a **real model identifier available to your account/server**. The historical identifier shown above is the code's configured default; availability is not guaranteed in the future. Do not commit `.env` to GitHub. The `.env.example` file contains placeholders and is safe to share only after checking it contains no real secrets.

To use a local model, install Ollama separately, start its service and pull the configured model (for example `ollama pull qwen2.5:7b`). Check connections with:

```bash
python check_backends.py
```

`check_backends.py` **attempts both Ollama and, when an API key exists, an OpenAI call**. It is not required to run a proxy experiment. An Ollama failure may be expected on a machine running only proxy tests.

## 3. Run a minimal smoke test — no live model needed

```bash
python run_experiment.py --episodes 2 --steps 2 --seed 7 --llm-mode proxy --systems full_l1_shell joint_evoswarm_cold joint_evoswarm --output-dir outputs_smoke
```

**Verified:** this command executed successfully against the ZIP's source in the documentation preparation environment. It wrote CSVs, learned-state files, metadata and plots. This establishes runnable code paths but is not evidence of learned generalisation. Some pandas versions print non-fatal `FutureWarning` messages.

## 4. Run standard proxy experiments

### Short complete 10-case manifest block

```bash
python run_experiment.py --episodes 10 --steps 6 --seed 7 --llm-mode proxy --output-dir outputs_proxy_10
```

This includes 4 experience, 3 recurrence and 3 compound generalisation episodes, per system. Use it for protocol checking, not publication-level statistics.

### Longer 120-episode proxy run

```bash
python run_experiment.py --episodes 120 --steps 6 --seed 7 --llm-mode proxy --output-dir outputs_proxy_120_new
```

The **current** 10-slot fixed manifest generates 48 experience, 36 recurrence, and 36 generalisation episodes for a 120-case run. This differs from some older notes describing 40/40/40. To verify, inspect:

```bash
python -c "import pandas as pd; x=pd.read_csv('outputs_proxy_120_new/scenario_manifest.csv'); print(x.groupby('phase').size())"
```

### Multi-seed proxy study

```bash
python run_multiseed.py --seeds 7 17 27 37 47 --episodes 60 --steps 6 --llm-mode proxy --output-dir outputs_multiseed_new
```

This runner creates `seed_7/`, `seed_17/`, etc., and at the root `per_seed_summary.csv`, `paper_summary_mean_std.csv`, `self_evolution_per_seed.csv` and `self_evolution_mean_std.csv` (where applicable). Standard deviations are across seed-level measurements. For genuine uncertainty intervals, document your inferential method and do not count dependent within-run episodes as independent replicates.

## 5. Run live-model experiments

Live mode accesses configured model backends as needed; it may take longer, fail on backend/network issues, and incur OpenAI charges. Validate availability and quota before increasing episodes. It is important to report model identifiers, timestamps, tokens, latency, fallback count and call patterns.

### Small live pilot

```bash
python run_experiment.py --episodes 10 --steps 6 --seed 7 --llm-mode live --output-dir outputs_live_pilot
```

### Explicit architecture comparison including pure L3

```bash
python run_experiment.py --episodes 10 --steps 6 --seed 7 --llm-mode live --include-pure-l3 --systems fixed_hybrid vertical_evoswarm joint_evoswarm_cold joint_evoswarm pure_l3 --output-dir outputs_live_compare_new
```

`--include-pure-l3` is relevant to enabling the pure-frontier-model baseline in live mode. Consult the source code for exception/fallback handling; do not label an output folder 'live OpenAI results' until the run's model calls and fallback logs have been verified.

### Limit to selected systems for debugging

```bash
python run_experiment.py --episodes 10 --steps 6 --seed 7 --llm-mode proxy --systems full_l1_shell expert_fixed_swarm joint_evoswarm --output-dir outputs_selected
```

If `joint_evoswarm_cold` is omitted, experienced-vs-cold self-evolution comparisons will not be available; the summary CSV may have no self-evolution rows. If `joint_evoswarm` is omitted, the learned joint-agent tables will not be exported.

## 6. Interpret and analyse outputs

```bash
python analyse_results.py --output-dir outputs_proxy_120_new
```

This script expects `summary.csv`, `scenario_summary.csv` and `phase_summary.csv`, and reads `self_evolution_summary.csv` if available. Its display targets must match the output schema of the code revision that produced the directory; some older saved outputs are incompatible with the current analysis script.

### Report at minimum

| Dimension | Recommended metrics / source |
|---|---|
| Continuous quality | `mean_operational_score`, `service_level`, `urgent_service`, backlog and operational cost |
| Scenario achievement | `scenario_achievement_score` for the current revision; legacy binary `scenario_success` separately |
| Causal adequacy | `causal_coverage`, `agent_recall`, active agents and executed agents |
| Topology | Communication edges, added/removed agents/edges, runtime expansions where exported |
| Resource proxies | `resource_cost`, decision units, L2/L3 calls; token and latency fields for live invocations |
| Self-evolution | Experienced-minus-cold operational/achievement improvement; cold-minus-experienced resource difference; recurrence/generalisation reuse |
| Variability | Independent seeds, mean ± standard deviation, corresponding manifest and model snapshot |

The synthetic `RESOURCE_COST` weight table in `evoswarm/experiment.py` assigns weights to specialist execution, shell calls, L2 calls, L3 calls and communication. It is not a measured USD cost, electricity cost, computational energy or wall-clock runtime. Likewise, `DECISION_UNITS` define the synthetic decision-work/deadline model. Keep quality, resource proxy and observed API latency separate in paper tables.

## 7. Experiment design and interpretation limits

Eight systems are available: `full_l1_shell`, `expert_fixed_swarm`, `fixed_hybrid`, `vertical_evoswarm`, `joint_evoswarm_cold`, `joint_evoswarm`, `pure_l2` and `pure_l3`. The joint architecture is designed to cover causally required capabilities without unconditionally using all eight specialists; cold joint is the relevant cross-episode learning control. A higher score in proxy mode is not proof of superiority with a real LLM.

The current `scenario_manifest.csv` was added to address consistency across episode counts. Some historical output folders in the uploaded ZIP (including a saved `outputs_proxy_120`) lack a manifest and use older names such as `scenario_success_rate`, where the current code exports `legacy_scenario_success_rate` and `scenario_achievement_score`. **Do not regenerate plots from mixed-version CSVs or claim the old 120-row run followed the current fixed manifest.** Prefer fresh runs under new output directory names and keep their metadata.

## 8. Troubleshooting

- **PowerShell prevents activation:** use `.\.venv\Scripts\python.exe` directly rather than changing machine-wide script policy.
- **`ModuleNotFoundError`:** confirm you installed `requirements.txt` into the interpreter that actually runs the script.
- **`Ollama FAILED`:** start the server, check port/endpoint, and confirm `OLLAMA_MODEL` is pulled. Proxy mode does not require it.
- **`OpenAI FAILED` or 401/404:** verify API key, account permissions, balance and actual model availability; never paste credentials into issues.
- **Slow or expensive live runs:** start with the 10-episode pilot, review the selected systems and L3 calls, then scale deliberately.
- **`KeyError` when analysing old CSVs:** the original ZIP includes historical output schemas; regenerate using the current code or process older results with their matching original revision.
- **`self_evolution_summary.csv` has no rows:** both `joint_evoswarm` and `joint_evoswarm_cold` must be present for the pairwise calculation.
- **Missing joint learned-state exports:** include `joint_evoswarm` in the system selection.
- **Proxy numbers differ:** check source revision, seed, run length, systems, step count, manifest and installed dependency versions. A fixed scenario *schedule* is not by itself a universal guarantee of identical numerical trajectories if RNG use changes.
- **Non-fatal pandas FutureWarning:** warnings in concatenation do not necessarily indicate failed output; check exit status and CSV files.

## 9. Reproducibility checklist before publishing

- [ ] Run commands in this document with the released code revision.
- [ ] Retain `run_metadata.json` and `scenario_manifest.csv` for each reported study.
- [ ] Record Python and pip versions: `python --version` and `python -m pip freeze > environment_freeze.txt`.
- [ ] State exactly which experiments used `proxy` versus `live`.
- [ ] Record actually used model IDs, API dates, local Ollama version, fallbacks, tokens and latency.
- [ ] Run and report multiple independent seeds, not a best-seed anecdote.
- [ ] Distinguish simulation resource-cost units from currency/energy measurements.
- [ ] Verify that reported tables correspond to current output column names and scenario protocol.
- [ ] Include a citation file/DOI and choose a code/data licence before release.
- [ ] Exclude `.env`, secret keys, virtual environments and `__pycache__` from the public repository.

