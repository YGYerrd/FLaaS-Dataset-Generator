# Catalogue and pool layout

Git excludes `outputs/`. A fresh checkout can generate a client catalogue with the [execution guide](../README.md#2-generate-a-client-catalogue).

| Prepared pool input | Use |
| --- | --- |
| `services.db` | Completed client records, metrics and provenance |
| `services.csv` and `pools.csv` | Verified service membership and verified family dependencies |
| `service_artifacts/` | Exact recipes, family documents, audit reservations, shared initial checkpoints and reconstruction support |
| `path_map.json` | Maps original absolute prefixes to local destinations after relocation |
| Model and dataset caches | Supply the pinned sources needed for local replay |

For the prepared classification dataset, place the bundle under `outputs/pooled/` and point the runtime at the matching caches. The composition manifest runner reads the path map beside its source database. Update resolved destinations in that map when moving machines and preserve the bytes of the signed family and recipe documents.

The legacy catalogue supports analysis of historical client measurements. For the published inventory, signature coverage, preparation checks and historical limitations, see the [original dataset inventory](https://github.com/YGYerrd/FLaaS-Dataset-Generator/blob/78e308240bf78cbd9bc59de2e2120b68cc64a3cd/docs/datasets.md).
