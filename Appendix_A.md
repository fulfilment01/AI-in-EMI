# Appendix A — Reproducibility materials (data lineage, pseudocode, configs, seeds, commands)

This Appendix documents the data lineage, the `requests.csv` specification (simulator input), high-level pseudocode to reconstruct requests from raw sources, example configuration & seeds, exact commands to reproduce traces/simulations/bootstrap, the expected outputs, and a concise file manifest. Follow these steps to reproduce the experiments reported in the manuscript.

---

## A1. Data lineage

- **Workload / event traces (primary)**  
  **Source:** Open University Learning Analytics Dataset (OULAD). Primary files used: `studentVle.csv`, `studentInfo.csv`.  
  **Purpose:** Provide empirical arrival processes (timestamps) and event/request types that drive the synthesized request traces.

- **Demographics (augmentation for equity slices)**  
  **Source:** UCI Student Performance dataset (`student-mat.csv` / `student-por.csv`).  
  **Purpose:** Provide demographic marginal distributions (sex, internet access, parental education) that are sampled and appended to synthesized requests to enable subgroup equity analyses.

> **Privacy note:** Raw OULAD and UCI datasets are not redistributed in this repository. The repository contains scripts to reconstruct `requests.csv` locally after the user downloads the raw datasets to `data/raw/`.

---

## A2. `requests.csv` (simulator input) — specification

Each row corresponds to one incoming request. Required columns:

- `timestamp` — numeric (seconds since epoch) or float relative time (chronological order).  
- `service_time` — float (seconds): estimated processing time for that request.  
- `request_type` — string (e.g., `vle_click`, `resource_download`).  
- `student_id` — anonymized identifier (string or integer).  
- `sex` — categorical (e.g., `M`/`F` or `male`/`female`).  
- `internet` — categorical (e.g., `yes`/`no`).  
- `medu` — parental education (numeric or categorical).  
- `fedu` — parental education (numeric or categorical).

Optional additional columns are ignored by the simulator but may be present for analysis.

---

## A3. High-level pseudocode — `scripts/reconstruct_requests.py`

```python
# reconstruct_requests.py (high-level pseudocode)

# Load raw files (local download required)
vle_df   = load_csv('data/raw/studentVle.csv')
info_df  = load_csv('data/raw/studentInfo.csv')
uci_df   = load_csv('data/raw/student-mat.csv')  # or student-por.csv

# 1. Filter/clean
vle_df = filter_course_terms(vle_df, course_ids, start_date, end_date)
vle_df = normalize_timestamps(vle_df, time_col='date')

# 2. Arrival model (preserve diurnal patterns)
arrivals_by_hour = aggregate_hourly_counts(vle_df['timestamp'])
interarrival_sampler = fit_empirical_interarrival(arrivals_by_hour)

# 3. Service-time models (per request_type)
service_models = {}
for rtype in unique(vle_df['request_type']):
    durations = extract_processing_durations(vle_df[vle_df.request_type==rtype])
    service_models[rtype] = fit_empirical_or_parametric(durations)

# 4. Demographic sampler (preserve marginals)
demographic_sampler = build_marginal_sampler(uci_df, ['sex','internet','medu','fedu'])

# 5. Synthesize requests
set_random_seed(SYNTHESIS_SEED)
requests = []
while len(requests) < SAMPLE_CAP:
    t = sample_next_timestamp(interarrival_sampler)
    rtype = sample_request_type(vle_df)          # maintain observed type ratios
    st = sample_service_time(service_models[rtype])
    demo = demographic_sampler.sample_one()
    requests.append({
        'timestamp': t,
        'service_time': st,
        'request_type': rtype,
        'student_id': anonymize_id(),
        **demo
    })

write_csv('data/requests.csv', requests)


## A4. Simulator usage — `scripts/run_simulation.py`

Purpose: read `data/requests.csv`, simulate N identical servers under a scheduling policy, and write per-request outcomes to `results/`.

Example CLI:
```bash
python scripts/run_simulation.py \
  --requests data/requests.csv \
  --policy least_loaded \
  --server-count 4 \
  --seed 101 \
  --out results/least_loaded_seed101.csv



### A5 — RL training & evaluation 
python scripts/run_rl.py --requests data/requests.csv --config configs/config.yml --episodes 500 --seed 101 --save-dir models/rl_hier_seed101


python scripts/eval_rl.py --model models/rl_hier_seed101 --requests data/requests.csv --out results/rl_hier_eval_seed101.csv


### A6 — Bootstrap procedure 
Procedure: resample per-request rows with replacement 1000 times per policy, compute mean RT, p95, and SLA% per resample, then report 2.5/97.5 percentiles.

Example:
python scripts/bootstrap_ci.py --results-dir results/ --n 1000 --out summaries/bootstrap_summary.csv

Output summaries/bootstrap_summary.csv columns: policy, metric, estimate, ci_low, ci_high


### A7 — Configs 
This repo uses configs/config.yml for experiment knobs. Example content (already in repo):

server_count: 4
SLA_seconds: 2.0
SAMPLE_CAP: 10000
SERVICE_EXP_SCALE: 0.5
BASE_SERVICE: 0.8
bootstrap_resamples: 1000

rl:
  learning_rate: 0.1
  discount: 0.99
  epsilon_start: 1.0
  epsilon_end: 0.01
  epsilon_decay_episodes: 1000
  episodes: 500


### A8 — Seeds 
Seeds are used for reproducibility. Example seeds.json (repo root):

{
  "synthesis_seed": 42,
  "sim_seed_list": [101, 102, 103, 104, 105],
  "bootstrap_seed": 2025
}
All scripts read seeds from this file (or accept --seed overrides) so runs are deterministic when needed


### A9 — Reproduction commands 
A9. Exact reproduction commands (step-by-step)

Prepare raw data (local): put OULAD & UCI CSVs into data/raw/.

Reconstruct requests:
python scripts/reconstruct_requests.py --config configs/config.yml --data-dir data/raw --out data/requests.csv

Run simulator (example for least_loaded):
python scripts/run_simulation.py --requests data/requests.csv --policy least_loaded --server-count 4 --seed 101 --out results/least_loaded_seed101.csv

Train RL (optional):
python scripts/run_rl.py --requests data/requests.csv --config configs/config.yml --seed 101 --save-dir models/rl_hier_seed101

Compute bootstrap CIs:
python scripts/bootstrap_ci.py --results-dir results/ --n 1000 --out summaries/bootstrap_summary.csv

Generate figures:
Open notebooks/analysis.ipynb and run cells; output saved to figures/.


### A10 — Expected outputs & manifest 

A10. Expected outputs and file manifest

Expected outputs (local):

data/requests.csv — synthesized trace

results/*.csv — per-run per-request CSVs

summaries/baseline_summary.csv — aggregated means

summaries/bootstrap_summary.csv — bootstrap estimates and CIs

figures/*.png — figures for manuscript

models/ — saved RL models

Repository manifest (should exist in repo root):
README.md, Appendix_A.md, configs/config.yml, seeds.json, scripts/*.py, notebooks/*.ipynb, requirements.txt, figures/, results/ (if not large), summaries/.


### A11 — Troubleshooting 
A11. Troubleshooting
If a script errors due to missing columns, inspect raw CSV headers and update mappings in scripts/reconstruct_requests.py.

If bootstrap CIs are unstable, increase bootstrap_resamples or SAMPLE_CAP and add more seeds.

For environment issues, install from requirements.txt or create a venv: python -m venv venv; source venv/bin/activate; pip install -r requirements.txt.