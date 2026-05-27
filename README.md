# Optimizing Gate Assignment and Tug Scheduling at ATL

A mixed-integer program for jointly assigning aircraft to gates and scheduling pushback tug operations at Hartsfield–Jackson Atlanta International Airport (ATL). The model accounts for gate-overlap constraints, gate-to-gate travel times, and service-time-feasible tug sequencing, and minimizes a weighted sum of remote-stand assignments and pushback lateness.

**MIT, December 2025 · Marc Nasr & Karl Chaker**

## What we built

We took a three-step modeling approach, starting simple and adding realism:

1. **Model 1 — Gate Assignment with Remote Stands.** Pure gate-assignment MILP with the option to send flights to higher-cost remote stands. Establishes the baseline tradeoff between gate scarcity and remote penalties.
2. **Model 2 — Full-Day Integrated Gate + Tug Scheduling.** Adds tug assignment, pushback timing, and travel-time-aware sequencing. Scales as O(|F|² |G|²) in binary variables — for 800 flights and 36 gates this blows up to ~10⁹ binaries and crashes Gurobi at build time.
3. **Model 3 — Scaled Integrated Model.** Restricts Model 2 to a busy 4-hour window: 35 flights × 10 gates × 6 tugs ≈ 1.2 × 10⁵ binaries. Solves to optimality in 76 minutes on a laptop.

## Key findings

From the Model 3 optimal solution:

- **Gate capacity is the binding constraint, not tug capacity.** 30/35 flights go to gates, 5 to remote stands — even with a remote penalty 200× the lateness penalty.
- **Tug utilization stays well below saturation.** Most tugs serve 4–7 flights; none approach infeasibility. Adding more tugs would not help while gates remain the bottleneck.
- **Travel-time-aware sequencing introduces essentially zero lateness.** All tug sequences remain time-feasible; all objective cost arises from the remote penalty.

## Mathematical formulation

**Decision variables**

| Variable | Meaning |
|---|---|
| `x[f,g] ∈ {0,1}` | flight `f` uses gate `g` |
| `y[f] ∈ {0,1}` | flight `f` sent to remote stand |
| `z[f,k] ∈ {0,1}` | tug `k` serves flight `f` |
| `u[f1,f2,k] ∈ {0,1}` | tug `k` serves `f1` before `f2` |
| `w[f1,f2,g1,g2] ∈ {0,1}` | linking: `f1`→`g1` and `f2`→`g2` |
| `s[f] ≥ 0` | actual pushback time |
| `L[f] ≥ 0` | pushback lateness |

**Objective**

```
min  C_remote · Σ y[f]  +  C_late · Σ L[f]
```

Calibrated `C_remote = 200`, `C_late = 1` — one remote assignment ≈ 200 minutes of pushback delay.

**Key constraint — travel-time-aware tug sequencing**

```
s[f2] ≥ s[f1] + SERVICE_TIME + Σ travel[g1,g2] · w[f1,f2,g1,g2]
            - M · (1 - u[f1,f2,k])
```

This is what makes the model honest — tugs can't teleport between gates.

## Data

- **Source:** [Kaggle Airline Dataset](https://www.kaggle.com/datasets/iamsouravbanerjee/airline-dataset) (derived from U.S. DOT On-Time Performance database). Not committed to this repo — download it separately.
- We extract scheduled arrival/departure times, airline ID, flight number, and tail number for ATL flights on a representative operational day.
- Synthetic inputs: 36 gates, gate-to-gate travel matrix `~ N(7, 2)` minutes, tug fleet of 3–6 vehicles, Bernoulli draws for remote-eligibility.

## Results visualizations

| File | What it shows |
|---|---|
| `gate_vs_remote.png` | Breakdown of gate-assigned vs. remote-stand flights |
| `gate_utilization.png` | Per-gate occupancy across the 4-hour window |
| `tug_load.png` / `tug_load_clean.png` | Per-tug task counts in the optimal solution |
| `cost_decomposition.png` | Objective decomposed by remote penalty vs. lateness |

The notebook also generates Gantt charts for both tug sequences and gate occupancy.

## Running

```bash
# 1. Get the dataset from Kaggle (link above) into the repo root
# 2. Install dependencies
pip install gurobipy pandas numpy matplotlib jupyter
# 3. Run the notebook
jupyter notebook PythonNotebookAirline.ipynb
```

Requires a Gurobi license (free academic license at gurobi.com).

## What we'd do with more time

1. **Scale to full-day operations** via decomposition, rolling-horizon, or column generation
2. **Calibrate against real ATL geometry** — current gate-to-gate matrix is synthetic
3. **Add stochasticity** — scenario-based or buffer-based robust formulations for arrival delays, weather disruption, and ground congestion

## Repo contents

- `PythonNotebookAirline.ipynb` — the full pipeline: data prep → Model 1 → Model 2 → Model 3 → visualizations
- `*.png` — solution plots
- See report PDF (in submission) for the full mathematical derivation and 9-page write-up
