# Run federated rounds from a fresh client catalogue

After [generating the example clients](../README.md#2-generate-a-client-catalogue), use five completed, replay-eligible classifiers to prepare a small local pool. This procedure verifies membership by actually replaying a five-client combination. It also collects one-shot timings for groups of two and three, so the round study has a baseline for each supported size. Run these examples from the repository root in PowerShell using a fresh output directory for the experiment.

## Select and replay five clients

List the families in your catalogue. Pick a classification family with at least five eligible clients and replace `FAMILY_ID` below. The following Python block reads its inventory and creates a one-shot manifest. The batch executor checks the combination ID against the hash of the family and sorted member IDs.

```powershell
flaas-compose list-families --db outputs/services.db --output outputs/family_services.csv

@'
import hashlib
import json
import pandas as pd

family = "FAMILY_ID"
clients = pd.read_csv("outputs/family_services.csv")
eligible = clients[
    (clients.family_id == family) & (clients.task_family == "classification")
    & (clients.composition_compatible == 1) & (clients.primary_metric_name == "accuracy")
]
ids = sorted(eligible.service_id.unique())[:5]
if len(ids) < 5:
    raise ValueError("Choose a family with at least five eligible accuracy-scored clients")
rows = []
for size in [2, 3, 5]:
    members = ids[:size]
    identity = json.dumps([family, members], separators=(",", ":"), sort_keys=True)
    rows.append({"combination_id": "comb_" + hashlib.sha256(identity.encode()).hexdigest(),
                 "family_id": family, "composition_size": size, "service_ids": json.dumps(members)})
pd.DataFrame(rows).to_csv("outputs/pilot_baseline.csv", index=False)
'@ | python -

flaas-compose run-manifest --file outputs/pilot_baseline.csv --source-db outputs/services.db --db outputs/pilot_baseline.db --device cpu --dry-run
flaas-compose run-manifest --file outputs/pilot_baseline.csv --source-db outputs/services.db --db outputs/pilot_baseline.db --device cpu
flaas-compose export-flaas --db outputs/pilot_baseline.db --source-db outputs/services.db --output outputs/pilot_members.csv
```

## Derive the pool from completed executions

Use the same source database and its original artefacts throughout. For this fresh catalogue, its parent directory (`outputs`) is the pool directory. The two CSV files below are the roster inputs for the batch sampler. The example writes them from measured results after checking that all three planned combinations completed.

```powershell
@'
import pandas as pd

plan = pd.read_csv("outputs/pilot_baseline.csv")
members = pd.read_csv("outputs/pilot_members.csv")
if set(members.combination_id) != set(plan.combination_id):
    raise ValueError("Check failed or missing baseline compositions before building the rosters")
services = members[["family_id", "client_id"]].drop_duplicates().rename(columns={"client_id": "service_id"})
services.assign(status="verified").to_csv("outputs/services.csv", index=False, mode="x")
pools = services.groupby("family_id").size().reset_index(name="services")
pools.assign(dependency_status="verified").to_csv("outputs/pools.csv", index=False, mode="x")
'@ | python -

flaas-compose generate-manifest --prepared-dir outputs --compositions 3 --rounds 10 --timing-db outputs/pilot_baseline.db --deadline-profile moderate --availability-probability 0.8 --seed 42 --output outputs/pilot_rounds.csv
flaas-compose run-manifest --file outputs/pilot_rounds.csv --source-db outputs/services.db --db outputs/pilot_rounds.db --device cpu --dry-run
flaas-compose run-manifest --file outputs/pilot_rounds.csv --source-db outputs/services.db --db outputs/pilot_rounds.db --device cpu
flaas-compose export-rounds --db outputs/pilot_rounds.db --output outputs/pilot_round_clients.csv
```

With three completed ten-round trajectories, the export contains 30 distinct combination-rounds and 100 client-round rows, including unavailable clients. See [the federated loop](../README.md#4-run-the-federated-loop) for the weighting, deadline and audit rules.
