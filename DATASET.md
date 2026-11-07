# Dataset and scenario documentation — EvoSwarm V1.4

## 1. Nature and purpose of the data

EvoSwarm is evaluated in a **procedurally generated supply-chain simulator**. The dataset is a collection of generated *episodes* (each involving a time-indexed operational state, events, agent decisions and consequences) rather than a downloaded real-world benchmark. It is designed to examine causal decision-making, delayed credit assignment, contextual trust, minimum-sufficient specialist selection, reasoning escalation and reuse under recurrence/compound change. The generator is `evoswarm/scenarios.py`; state dynamics and outcome computation are in `evoswarm/environment.py`; logged evaluation is in `evoswarm/experiment.py`.

Each episode is a separate scenario instantiation. At each step the environment provides a state and event context, the system selects experts and an action (e.g. procurement quantity/supplier, staff overtime, urgent priority, transport acceleration or forklift rental), and the simulator applies dynamics to determine outcomes including fulfilled demand, urgent fulfilment, remaining backlog, stock/inventory, cost and decision deadline.

**No external corporate records or patient/human-subject data are asserted to be present.** The generated demand figures, risks, outcomes and trust evidence are synthetic. They should not be presented as measurements from a real business.

## 2. Generation mechanics and reproducibility

The code defines `ScenarioDefinition` records, each holding base demand/urgency multipliers, supplier and route risk values, staff/equipment constraints, time budget, initial semantic events, `StepChange` timeline updates and a causal signature. `make_scenario()` samples variable quantities using `random.Random`; `choose_scenario()` selects the episode's family/phase through `_manifest_spec()`.

The current fixed manifest constant is `v14-fixed-block10-r1`, with `MANIFEST_BLOCK_SIZE = 10`. For each consecutive 10-episode block the selection is:

| Block slot (zero-based) | Phase | Scenario family assignment |
|---|---|---|
| 0 | experience | `normal_replenishment` |
| 1–3 | experience | three rotating challenge families |
| 4–6 | recurrence | the same three families, with altered surface scenario details |
| 7–9 | generalisation | three rotated compound families |

The challenge families rotate through the six non-normal base families; compound families rotate among three choices. Consequently, for episode counts divisible by 10, the phase distribution is **40% experience, 30% recurrence, 30% generalisation**. The episode-to-family specification depends on *absolute episode number*, not the final `--episodes` argument. The 10-case prefix of a larger run therefore has the same scheduled families; exact resulting numerical states additionally depend on seed and RNG-consumption/order. Always retain the generated `scenario_manifest.csv` to verify actual cases and variants.

### Worked example: first ten episodes

With the first block's rotation (`block=0`), the scheduled family sequence is:

| Episode | Phase | Family |
|---:|---|---|
| 1 | experience | `normal_replenishment` |
| 2 | experience | `supplier_disruption` |
| 3 | experience | `staff_capacity_shift` |
| 4 | experience | `promotion_capacity` |
| 5 | recurrence | `supplier_disruption` |
| 6 | recurrence | `staff_capacity_shift` |
| 7 | recurrence | `promotion_capacity` |
| 8 | generalisation | `compound_supplier_staff` |
| 9 | generalisation | `compound_demand_transport` |
| 10 | generalisation | `compound_multi_constraint` |

The variant suffix identifies a base/analogue/novel compound variant, with an index derived from the episode number. Sampled operational values are stochastic: do not assume a specific risk/demand number from this table.

## 3. Scenario catalogue

| Family | Why included / causal stress |
|---|---|
| `normal_replenishment` | Standard demand/inventory/procurement decisions; reference operating regime |
| `supplier_disruption` | Supplier reliability shock, inventory risk, and later recovery |
| `staff_capacity_shift` | Unexpected absences, warehouse-capacity constraint and later staffing recovery |
| `promotion_capacity` | Demand/urgent-demand increase and staff constraint; hidden demand-agent bias tests contextual trust |
| `conflicting_supplier_transport` | Supplier availability versus transport-route risk; integrates contradictory-looking evidence |
| `false_alarm_then_shift` | Initial weak supplier warning, retraction and subsequent real staffing bottleneck |
| `time_critical_dispatch` | Tight decision budget requiring operational/reasoning trade-offs |
| `compound_supplier_staff` | Novel combination of supplier and staffing disruption |
| `compound_demand_transport` | Combined demand and transport constraint |
| `compound_multi_constraint` | Several simultaneous causal pressures, potentially including equipment failure |

**Note on hidden variables:** The generator may set `latent_agent_bias` for selected specialists/regimes. `WorldState` describes this explicitly as a hidden simulation variable excluded from compact/full World Model views, so methods cannot simply read the designed error directly. This is a controlled test of learning contextual trust, not evidence of naturally occurring bias in a real operating system.

## 4. Data lifecycle and hierarchy

1. **Scenario specification**: family, phase, variant, initial risk/capacity parameters and scheduled semantic events.
2. **Episode initialization**: the environment samples an initial operating state and initializes demand, inventory, supplier/transport and staffing variables.
3. **Decision step**: the world model exposes available state, time/history, observed events and causal indications to the agent system.
4. **Agent decision**: activated specialists exchange evidence and produce a structured operational action; L2/L3 may be used depending on policy/mode.
5. **Outcome and feedback**: environment computes fulfilment, cost, backlog, inventory and deadline consequences; adaptive methods obtain immediate and delayed feedback.
6. **Experience**: the experienced joint system stores a compact causal capsule containing context signature, agents used, useful agents, reasoning level and outcome/resource summary.
7. **Exports**: step-level, episode-level, aggregated and learned-state files are written to the chosen `--output-dir`.

## 5. Variables and example interpretation

The principal state groups are:

- **Demand and urgency:** `current_demand`, `current_urgent_demand`, `demand_forecast`, demand/urgent multipliers.
- **Inventory and backlog:** `inventory`, `backlog`, `urgent_backlog`, inbound orders and delays.
- **Supplier and transport:** supplier A/B risk and lead time, route A/B risk, transport capacity and restrictions.
- **Staff and warehouse:** available employees, throughput per worker, forklift availability and resulting throughput constraints.
- **Events and uncertainty:** event type, target, natural-language evidence text, severity, confidence, source, expiry and age.
- **Time and budget:** `episode`, step index, scenario/phase and decision budget units.

### Illustrative semantic event (structure, not an exact output record)

```json
{
  "event_type": "supplier_disruption",
  "target": "supplier_A",
  "text": "Severe weather is disrupting Supplier A outbound logistics.",
  "severity": 0.30,
  "confidence": 0.90,
  "source": "supplier_bulletin",
  "expires_in": 3,
  "age": 0
}
```

The event's wording and fields come from the code's `SemanticEvent` model and supplier-disruption generator; values are an illustrative formatting example. The text is synthetic evidence presented to decision systems, not real supplier news.

### Step-level example (interpretation)

A `step_results.csv` row identifies a particular `system`, `episode`, `scenario`, `phase` and `step`, then records the observed causal `features`, semicolon-delimited `active_agents`, action fields such as `order_qty`, `supplier`, `overtime_workers`, `urgent_priority`, `expedite_transport` and `rent_forklift`, and resulting `service_level`, `urgent_service`, `backlog_end`, `operational_score`, `resource_cost` and `deadline_met`. Each row is a **simulation decision/outcome observation**, not a static classification label.

## 6. Output-file dictionary

A fresh run of the uploaded code writes the following into `--output-dir`:

| File | Unit / interpretation |
|---|---|
| `scenario_manifest.csv` | One unique episode-to-phase/scenario/variant mapping; use to audit evaluation composition |
| `step_results.csv` | One row per system × episode × simulation step; operational action/outcome and agent metrics |
| `episode_results.csv` | One row per system × episode; whole-episode quality, resource and adaptation metrics |
| `summary.csv` | One row per system; cross-episode means and totals |
| `scenario_summary.csv` | One row per system × scenario family; family-specific mean outcomes |
| `phase_summary.csv` | One row per system × phase; experience/recurrence/generalisation comparison |
| `self_evolution_summary.csv` | Experienced-minus-cold quality metrics and cold-minus-experienced resource saving per phase; empty if both are not selected |
| `joint_agent_qtable.csv` | Learned topology/agent-level policy values from experienced joint system |
| `joint_pair_qtable.csv` | Learned communication/agent-pair policy values |
| `joint_level_qtable.csv` | Learned choice of reasoning levels |
| `contextual_agent_trust.csv` | Context-conditioned specialist reliability estimates |
| `causal_capability_graph.csv` | Feature-to-agent capability edge weights |
| `causal_agent_pair_graph.csv` | Agent-pair associations/synergies |
| `experience_memory.csv` | Compact retrieved/reusable experience-capsule fields |
| `run_metadata.json` | Protocol, code version, model settings, manifest and system identifiers |
| `*.png` | Plots generated by `evoswarm/plots.py` |

Learned-state files are available when `joint_evoswarm` is run. Their absence when selecting only another baseline does not mean the experiment failed.

### Key CSV fields

| Name | Location(s) | Meaning |
|---|---|---|
| `system` | steps, episodes, summaries | Baseline/algorithm identifier |
| `episode`, `step` | step results | Episode ID and within-episode time index (zero-based step) |
| `phase`, `scenario`, `scenario_variant` | steps, episodes, manifest | Intended experience regime, family and instance |
| `active_agents` | step results | Semicolon-delimited specialist identifiers selected for the decision |
| `active_agent_count` | step results | Number of active specialists |
| `agent_execution_count` | step results | Total specialist executions (may exceed unique active agents due to refresh/re-execution) |
| `causal_coverage` | step results | Fraction of required causal capabilities covered by activated team |
| `agent_precision`, `agent_recall` | step results | Coverage-oriented agent-selection diagnostics (not predictive classification precision/recall) |
| `service_level`, `urgent_service` | steps | Fraction of demand/urgent demand fulfilled in that simulated step |
| `backlog_end`, `inventory_end` | steps | State remaining after action |
| `operational_score` | steps | Continuous weighted quality summary, in the code's 0–100 convention |
| `resource_cost` | steps | **Synthetic weighted intelligence expenditure**, not real currency or energy |
| `decision_units`, `decision_budget_units` | steps | Proxy timing/work budget and constraint |
| `deadline_met` | steps | Whether proxy decision work complied with the step's budget |
| `scenario_success` | episodes | Strict legacy binary episode outcome; do not treat as sole rank metric |
| `scenario_achievement_score` | episodes, current-version summaries | Continuous episode-level scenario achievement metric |
| `mean_operational_score` | episodes, summaries | Aggregated operational quality |
| `mean_resource_cost` | episodes | Average synthetic resource proxy for the episode |
| `shell_calls`, `l2_calls`, `l3_calls` | episodes, summaries | Reasoning operations/calls by level |
| `input_tokens`, `output_tokens`, `llm_latency_ms` | episodes, summaries | Logged token and latency use; interpret differently in proxy vs live mode |
| `experience_retrieval_rate`, `experience_reuse_accuracy` | episodes/summaries | Frequency/usefulness of retrieved capsules |
| `adaptation_regret` | episodes/summaries | Internal adaptation quality indicator; see source implementation |

The exact generated columns can be inspected with `python -c "import pandas as pd; print(pd.read_csv('outputs_quickstart/episode_results.csv').columns.tolist())"`. This is particularly important because **older included output folders do not contain all current columns**.

## 7. Data splits, leakage boundaries and limitations

- **No pre-existing train/test CSV split:** instead, the experiment runs a temporal schedule of experience, causal recurrence and compound generalisation episodes. The regimes must not be described as an independent random IID train/validation/test partition.
- **Recurrence is designed, not necessarily out-of-distribution by every formal definition:** repeating a causal family under changed surface details tests transfer of context-specific experience.
- **Generalisation is compound synthetic recombination:** it does not demonstrate transfer to an external real-world domain.
- **Hidden specialist perturbations:** biased specialist conditions are intentionally present in the simulator but not exposed as world-model inputs. Maintain this separation if editing the code.
- **Potential within-run dependence:** experience memory and policy are updated sequentially; successive episodes are **not independent** observations for statistical testing. Use **independent random seeds/runs** as the main unit of variability.
- **Comparison validity:** check that each compared method receives the protocol's intended information and that model access and budget are described. `pure_l2` and `pure_l3` are distinct baselines, not identical computational budgets.
- **Version boundary:** legacy saved results have schema differences; do not mix old and new CSV files when constructing tables without normalizing and checking their protocols.

## 8. Data availability and regeneration

The `outputs_*` directories in the original ZIP contain saved result data and charts. The original *source* of generated episodes is the Python generator, not a stand-alone static input file. Recreate experiment logs by running `run_experiment.py` or `run_multiseed.py` and commit the corresponding `scenario_manifest.csv` and `run_metadata.json` for every reported experiment. Capture dependency versions and model snapshots for live results.

For exact commands, see [EXPERIMENTS.md](EXPERIMENTS.md).
