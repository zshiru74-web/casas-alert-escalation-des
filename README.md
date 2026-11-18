# Responsibility-based alert escalation for older adults living alone

Code for the paper *"Responsibility-Based Alert Escalation for Older Adults Living Alone: A Trace-Driven
Discrete-Event Simulation Using Smart-Home Sensor Data"*.

The pipeline turns the ambient-sensor recordings of 24 single-resident CASAS homes into causal alert streams,
injects falls requiring assistance, and simulates three alert-response policies in a 600-home catchment with
capacity-limited community responders:

* **S0** – alert-only message to the family;
* **S1** – designated family responder with acknowledgement;
* **S2** – responsibility-based framework: verification call (C1), designated responder (C2), escalation to a
  community responder (C3), tiering and night-time routing (C4), and outcome-informed personal thresholds (C5).

## Data

The sensor data are not redistributed here. Download them from Zenodo:

| Release | DOI | Used in |
|---|---|---|
| CASAS raw release (39 homes, 147.0 million events, 2009–2021) | [10.5281/zenodo.16343844](https://doi.org/10.5281/zenodo.16343844) | step 1 |
| CASAS mapped release (sensor-to-room labels) | [10.5281/zenodo.15708568](https://doi.org/10.5281/zenodo.15708568) | step 2 (read remotely) |

Place the zip archives of the raw release in `data/raw/`. Step 2 reads the mapped `data.zip` member by member
with HTTP range requests and needs internet access.

## Requirements

Python 3.11. Install the packages with

```
pip install -r requirements.txt
```

The results in the paper were produced with Python 3.11.18, NumPy 2.4.6, pandas 2.3.3, SciPy 1.17.1, and
pyarrow 26.0.1 on Windows 11 (16 logical processors, 31 GB RAM).

## Running the pipeline

Run the scripts from the repository root, in order:

| Step | Command | Output | Paper |
|---|---|---|---|
| 1 | `python scripts/01_convert_raw.py` | `data/homes/`, `data/home_index.csv`, `data/sensor_inventory.csv` | Section 2.2 |
| 2 | `python scripts/02_sensor_map.py` | `data/sensor_map/`, `data/sensor_map_final/` | Section 2.1 |
| 3 | `python scripts/03_activity_episodes.py` | `data/activity/`, `data/episodes/`, `data/daily_qc.parquet`, `data/home_inclusion.csv` | Section 2.2 |
| 4 | `python scripts/04_alerts_fall_library.py` | `data/derived/` (alert streams, fall libraries, incl. `fall_library_day.parquet` with eight candidate falls per evaluation day) | Sections 2.3–2.4 |
| 5 | `python scripts/05_run_experiments.py` | `results/<experiment>/` | Sections 2.6–2.8 |
| 6 | `python scripts/06_analyse.py` | summary tables in `results/` (incl. responder accounting, Monte Carlo standard errors, GSA direction, Tables S2–S3) | Tables 3–5, S2–S3, Section 3 |
| 7 | `python scripts/07_figures.py` | `figures/Figure1–8.png/.pdf` | Figures 1–8 |

Approximate run times on a 16-core desktop: steps 1–2, 10–20 min (depending on the network); step 3, 1–2 min;
step 4, under 1 min; step 5, about 35 min with 14 parallel processes; steps 6–7, under 1 min.

## Code structure

| File | Content |
|---|---|
| `src/pipeline_events.py` | presence inference, alert rules E1–E3, fixed/generic/personal thresholds, fall detection delay (Equations (1)–(5)) |
| `src/des_sim.py` | catchment construction (falls aligned with the replayed day), response logic of S0–S2 and ablations, priority queue of community responders with optional day/night schedules, overtime and 06:00 backlog (Equations (6)–(9), Algorithms 1–2) |
| `src/convert_casas.py`, `src/fetch_map.py` | workers for steps 1 and 2 |
| `src/run_batch.py` | replications of the main experiment, home-resampled experiment, sweeps, responder schedules, decomposition variants, and detection sensitivity |
| `src/build_derived.py` | alert streams for alternative detection settings |
| `src/gsa.py` | Latin hypercube design, runs of the global sensitivity analysis, and selection of sets for additional replications |
| `src/warmup_check.py`, `src/informal_discovery.py` | initialisation-bias check and informal-discovery sensitivity analysis |
| `src/sim_extra.py` | stratified start offsets, residual benign alerts during fall episodes, duplicated start offsets |

## Reproducibility

All random numbers come from NumPy generators with fixed seeds. Replication *r* of the main experiment uses
`numpy.random.default_rng([20261116, r])`; the global sensitivity analysis uses `default_rng([99, set, r])` and a
Latin hypercube design from `scipy.stats.qmc` with seed 7. Common random numbers are used across scenarios. Detection configurations are processed in a fixed
(sorted) order, so that results do not depend on Python's hash randomisation.

Model options in `des_sim.BASE`: `align_falls` (draw each fall from the day that the virtual home replays),
`drop_after_fall_h` (remove the fallen resident's benign alerts for 24 h), and `unified_timing` (device alerts
skip the check call in every policy; day and night judged at the notification time). The final model uses all
three; step 5 also runs the variants V0 and V1 used for the paired check in Tables S2–S3.
The model is reported following the STRESS-DES guideline (Table S1 of the paper).

## Citation

If you use this code, please cite the paper (reference to be added after publication) and the CASAS datasets
listed above.

## License

MIT License (see `LICENSE`). The CASAS data are subject to the terms of their Zenodo records.
