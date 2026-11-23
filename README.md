# Federated Learning as a Service (FLaaS) Dataset Generator

Generate client catalogues and measure how groups of compatible clients perform under Federated Averaging (FedAvg). A **family** shares an initial checkpoint, tensor structure, labels, preprocessing and evaluation contract. Clients within it can have different training examples, class distributions and local training budgets.

Choose the workflow that matches your inputs.

| Starting point | Workflow | What you can measure |
| --- | --- | --- |
| Fresh checkout | [Generate a client catalogue](#2-generate-a-client-catalogue) | Each client's quality, training time and resource use |
| Catalogue with eligible client recipes | [Search and validate selections](#6-optional-search-and-validate-client-selections) | Predicted versus measured performance of selected groups |
| Verified classification pool | [Run one-shot compositions](#3-measure-one-shot-compositions) | Global accuracy for each client combination and a timing baseline |
| Same pool and completed one-shot timings | [Run the federated loop](#4-run-the-federated-loop) | How accuracy, participation and deadline violations change across rounds |

## 1. Install

Use Git and Python 3.12. Run the following commands in Windows PowerShell. The CPU setup also supports the documented composition commands on a machine without a GPU.

```powershell
git clone https://github.com/YGYerrd/FLaaS-Dataset-Generator.git
cd FLaaS-Dataset-Generator
py -3.12 -m venv .venv
. .\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
python -m pip install -r requirements.txt
python -m pip install --no-deps -e .
flaas-generate --help
flaas-compose --help
```

On Linux, create the environment with `python3.12 -m venv .venv` and activate it with `source .venv/bin/activate`. The example CLI commands below also work in Bash when joined onto one line. Install the appropriate PyTorch build first for CUDA or Linux ROCm. Windows ROCm has a separate [setup and activation procedure](docs/usage.md#python-and-platform-prerequisites). Initial training needs access to the selected model and dataset, or their pinned local caches.

The installed commands are equivalent to `python -m flaas_data_generator.cli.main` and `python -m flaas_composition`. When updating an existing checkout, reinstall the editable package and update scripts, imports and environment variables to the names shown here.

## 2. Generate a client catalogue

Plan ten text-classification clients from one model/dataset pair. A client samples a local partition and trains independently from the shared family checkpoint. The generator supports IID, Dirichlet and shard partitions.

```powershell
flaas-generate hf-manifest --task-keys text_classification --training-regimes finetune_transfer --models-per-task 1 --datasets-per-model 1 --clients-per-family 10 --average-samples 128 --heterogeneity-intensity 0.5 --seed 42 --evaluation-seed 42 --metric-evaluation-profile benchmark --max-train-time-s 3600 --max-eval-time-s 600 --output outputs/clients.csv
flaas-generate run-manifest --file outputs/clients.csv --dry-run
flaas-generate run-manifest --file outputs/clients.csv --db outputs/services.db
```

Review `clients.csv` and keep it with `clients.families.json`. The dry run resolves the planned rows. The final command trains and evaluates them, writes measured records to SQLite and retains the recipes and shared artefacts needed for replay. Repeating that command skips completed service IDs. Time budgets can interrupt training and affect replay eligibility.

| Setting | What it controls |
| --- | --- |
| `--clients-per-family` | Number of client recipes per compatible model/dataset contract |
| `--average-samples` | Centre of the local training sample counts |
| `--heterogeneity-intensity` | Degree of distribution skew for non-IID partitions, from 0 to 1 |
| `--seed`, `--evaluation-seed` | Repeatable registry and recipe selection, and evaluation sampling |
| `--metric-evaluation-profile` | Budget for behavioural metrics such as trust and explainability |
| `FLAAS_OUTDIR`, `--db` | Output root for generated artefacts and explicit catalogue path |

For the available tasks, run `flaas-generate task-capabilities`. The [full usage guide](docs/usage.md) covers device selection, worker limits, retention and manifest options.

## 3. Measure one-shot compositions

**Input for sections 3 and 4:** a prepared classification pool containing `pools.csv`, `services.csv`, `services.db`, client recipes, shared checkpoints, audit reservations and the pinned model/data dependencies. The examples use `outputs/pooled/`. Git excludes this directory. For a fresh catalogue from section 2, [prepare and run a five-client pilot](docs/local-classification-pool.md) to create the rosters and timing evidence from your own completed clients. For the archived classification study, restore the prepared bundle or rebuild it from the audited classification roster and original Lambda shards with [the preparation script](scripts/prepare_classification_pools.py). See the [pool layout and dependencies](docs/datasets.md).

Choose one verified family with at least five clients. This PowerShell example takes the first matching row from the prepared roster.

```powershell
$pool = 'outputs/pooled'
$family = (Import-Csv "$pool/pools.csv" | Where-Object { $_.dependency_status -eq 'verified' -and [int]$_.services -ge 5 } | Select-Object -First 1).family_id
flaas-compose generate-manifest --prepared-dir $pool --family-id $family --compositions 5 --seed 42 --output outputs/oneshot.csv
flaas-compose run-manifest --file outputs/oneshot.csv --source-db "$pool/services.db" --db outputs/oneshot.db --device cpu --dry-run
flaas-compose run-manifest --file outputs/oneshot.csv --source-db "$pool/services.db" --db outputs/oneshot.db --device cpu
flaas-compose export-flaas --db outputs/oneshot.db --source-db "$pool/services.db" --output outputs/oneshot_clients.csv
```

In Bash, set `pool=outputs/pooled` and `family` to the selected ID from `pools.csv`, then run the four CLI commands. The dry run checks the metadata and artefact references. Execution reconstructs each selected client, weights its contribution by realised local sample count, and evaluates the combined model on the family's reserved audit examples. The source catalogue and composition results use separate databases.

One-shot group sizes are 2, 3, 5, 7 and 10 where the family is large enough. The requested count is approximate because the sampler allocates a quota to each eligible family/size pair. Completed timings for sizes 2, 3 and 5 feed section 4.

## 4. Run the federated loop

Use the same pool, family and hardware as the one-shot baseline. This example requests one combination of each eligible size (2, 3 and 5), ten communication rounds, a moderate deadline and 80% availability per client per round. Each combination follows its own trajectory.

```powershell
flaas-compose generate-manifest --prepared-dir $pool --family-id $family --compositions 3 --rounds 10 --timing-db outputs/oneshot.db --deadline-profile moderate --availability-probability 0.8 --seed 42 --output outputs/rounds.csv
flaas-compose run-manifest --file outputs/rounds.csv --source-db "$pool/services.db" --db outputs/rounds.db --device cpu --dry-run
flaas-compose run-manifest --file outputs/rounds.csv --source-db "$pool/services.db" --db outputs/rounds.db --device cpu
flaas-compose export-rounds --db outputs/rounds.db --output outputs/round_clients.csv
```

1. **Initialise the global model.** Round 1 starts from the verified family checkpoint. Later rounds start from the global state produced by the previous round.
2. **Train available clients.** Each available client starts from the same round parent and reuses its recorded local examples and training settings. The runner processes clients sequentially in service-ID order. Availability comes from the seeded schedule frozen in the manifest.
3. **Apply the deadline.** Training starts only while time remains. A client already training can finish late. Late updates receive zero weight. The deadline is 0.75, 1.00 or 1.50 times the median one-shot duration for that family and size (tight, moderate or relaxed), after subtracting optional client-loss measurement time.
4. **Aggregate accepted updates.** For each accepted client, the FedAvg weight is its local sample count divided by the sum across accepted clients. One contributor can update the global state. If none contribute, the runner carries the parent forward and records `round_status=no_update`.
5. **Evaluate and repeat.** The runner measures global accuracy on the same held-out audit partition and records elapsed time, accepted updates and state hashes. Aggregation and evaluation can finish after the deadline. Updates pass in-process, so the elapsed time measures local sequential execution and excludes network transmission.

Leave out `--device cpu` to use the composition executor's automatic GPU selection on a supported host. A worker owns a complete trajectory. Completed trajectories are skipped on rerun. An interrupted trajectory restarts at round 1 because intermediate global checkpoints are not persisted. See the [round experiment guide](docs/composition-rounds.md) and [loop implementation](flaas_composition/round_execution.py).

## 5. Read the results

| Output | Unit of observation | Example use |
| --- | --- | --- |
| `outputs/services.db` | One service with related metric and provenance records | Compare local accuracy by partition strategy |
| `outputs/oneshot_clients.csv` | One member of a completed one-shot composition | Compare measured global accuracy by group size |
| `outputs/round_clients.csv` | One member of a completed communication round, including unavailable clients | Compare availability with `update_accepted` and `aggregation_weight` |
| `outputs/rounds.db` | Durable composition and round records | Inspect partial histories and distinguish failed trajectories from completed ones |

Round accuracy and duration repeat on every member row. To give a five-client group the same influence as a two-client group, reduce to one row per round before summarising. For example, plot one accuracy curve per trajectory from this table:

```python
import pandas as pd

members = pd.read_csv("outputs/round_clients.csv")
rounds = members.drop_duplicates(["run_id", "combination_id", "communication_round"])
print(rounds[["combination_id", "communication_round", "global_accuracy_per_round",
              "round_completion_time_s", "deadline_violation"]].head(10))
```

Use `export-flaas` for one-shot results and `export-rounds` for trajectory history. Missing measurements have distinct meanings, including unavailable transfer telemetry. See the [flat export definition](docs/flaas-export.md) and [round attributes](docs/composition-rounds.md#round-attributes).

## 6. Optional: search and validate client selections

With a completed recipe-backed catalogue from section 2, list the eligible families. Replace `FAMILY_ID` with a printed family ID with at least four eligible clients for the request profile below.

```powershell
flaas-compose list-families --db outputs/services.db --output outputs/family_services.csv
flaas-compose generate-requests --db outputs/services.db --family-id "FAMILY_ID" --profile recipe_composition_v6 --compatibility-mode strict --requests 1 --seed 42 --output outputs/requests.json
flaas-compose evaluate --db outputs/services.db --requests-file outputs/requests.json --output-dir outputs/search --search-budget 100 --seed 42
flaas-compose validate-search --db outputs/services.db --requests-file outputs/requests.json --evaluation-dir outputs/search --output-dir outputs/validation --device cpu
```

Search scores subsets using catalogue evidence. Validation replays the frozen selections and measures the aggregate on the reserved audit data. Compare predictions with measurements in `validation_comparison.csv`. The [composition guide](docs/recipe-composition.md) covers individual selections, audit sample requirements and validation failures.

## Troubleshooting and reproducibility

- **Missing family companion or replay file.** Restore the matching manifest companion or original recipes, checkpoints and pinned caches. For a relocated prepared pool, update the resolved destinations in `path_map.json` beside its source database.
- **No eligible family/size or timing baseline.** Check the verified member count in the rosters, then check for completed one-shot results for the same family and size. Reasons for skipped pairs appear in the manifest metadata.
- **Comparing deadlines.** Keep the pool, composition count, random seed, availability probability, hardware and timing database fixed. Write each deadline profile to a fresh manifest and results database.
- **Reproducing a run.** Retain the source revision, environment, hardware, seeds, manifests and companions, database shards, recipes, shared artefacts, audit reservations and exports. A seed fixes the sampled plan. Training time and numerical replay can vary by device.

Run `python -m pytest -q` for the regression suite. To exercise the federated loop with small test fixtures, run `python -m pytest -q tests/composition/test_rounds.py tests/generator/test_round_initial_state.py`. Further references are indexed in [docs](docs/README.md).
