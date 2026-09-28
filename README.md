# AAAC — Access-Aware Admission Control

A virtual waiting room that stays fair to users on slow connections.

Conventional waiting rooms give every user the same admission window and the same page. Users on slow, lossy links time out, go to the back of the queue, and retry at the same cost, so they are systematically excluded. AAAC adapts the queue to each client's link to close that gap.

The key metric is the **completion gap**:

```
Δ = completion_rate[HIGH] − completion_rate[LOW]
```

## How it works

| | Component | What it does |
|---|---|---|
| C1 | Link estimation | Classifies each client as HIGH / MEDIUM / LOW from its own traffic (LightGBM) |
| C2 | Adaptive window | Slower links get more time to finish |
| C3 | Payload adaptation | Slower links get a lighter page with the same content (411 KB → 5 KB → 2 KB) |
| C4 | Non-regressive re-queue | A timed-out client keeps its place and gets a cheaper retry |
| C5 | AIMD rate control | Admission rate tracks origin health so the server never collapses |

**Services:** admission (`:8000`, with Redis), delivery (`:8001`), mock origin (`:8002`), plus three network-shaped client containers (HIGH / MEDIUM / LOW via `tc netem`).

**Run modes** (set in `configs/run.yaml`): `none` (no queue), `baseline` (conventional waiting room), `aaac` (full system).

## Quick start

Requires Python 3.11 and Docker (OrbStack recommended — ingress shaping needs `ifb` support).

```bash
make install                 # venv + dependencies
make check                   # lint, typecheck, tests

# build the classifier (not committed)
PYTHONPATH=src .venv/bin/python -m aaac.estimator.train --n 12000 --seed 1

cp .env.example .env         # set AAAC_TOKEN_SECRET

make up RUN_ID=s1-aaac       # start the testbed
make verify-testbed          # confirm network shaping works
make experiment              # run all modes × seeds
make report                  # generate results report
make down
```

Run `make help` for all targets.

## Results

3 seeds × 3 modes, 140 clients per run:

| Mode | Mean Δ | Aggregate completion |
|---|---:|---:|
| `none` | 0.823 | 0.712 |
| `baseline` | 0.986 | 0.655 |
| `aaac` | **0.027** | **0.991** |

Under the baseline, almost no low-bandwidth clients finished; under AAAC nearly all did, with zero origin errors and faster completion for high-bandwidth clients. The pre-registered hypothesis was supported (paired difference 0.959, 95% CI [0.871, 1.047], p = 0.0005).

Limitations: only 3 seeds (5 planned), a scaled-down load, and the classifier was rarely exercised in these runs. Full details in [`results/report-final.md`](results/report-final.md).

## Project structure

```
src/aaac/
├── common/       # schemas, tokens, event log, config
├── admission/    # queue, window, controller, re-queue
├── estimator/    # link classifier
├── delivery/     # page variants
├── client/       # client SDK
├── dashboard/    # live dashboard
├── origin/       # mock origin
└── evaluation/   # testbed, load gen, metrics, report
```

More documentation in [`docs/`](docs/).

## Team

- **Thisaru Ramanayaka** — admission, controller, re-queue
- **Sachintha Dilshan** — estimator, delivery, client SDK, dashboard
- **Devmith Amarasekara** — origin, testbed, evaluation

## License

MIT
