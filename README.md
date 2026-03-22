# collection-market-tracker-data

The data layer for the **Collection Market Tracker** — a **BigQuery + Cloud Run + GCS + Git** architecture that keeps data permanently accessible even when GCP billing is disrupted, by treating this Git repository as the source of truth.

## Related repositories

| Repo | Description |
|---|---|
| [collection-market-tracker-backend](https://github.com/FutureGadgetCollections/collection-market-tracker-backend) | Populates this repo with scraped/processed market data |
| [collection-market-tracker-frontend-admin](https://github.com/FutureGadgetCollections/collection-market-tracker-frontend-admin) | Admin UI that reads from BigQuery / Cloud Run backed by this data |

## Architecture overview

```
┌────────────────────────────────────────────────────────────┐
│  collection-market-tracker-backend                         │
│  Scrapes / processes market data, commits to this repo     │
└────────────────────────┬───────────────────────────────────┘
                         │ git push
┌────────────────────────▼───────────────────────────────────┐
│  collection-market-tracker-data  (this repo)               │
│  data/*.jsonl  ←  source of truth for all records          │
└────────────────────────┬───────────────────────────────────┘
                         │ GitHub Actions (on push to main)
          ┌──────────────▼──────────────┐
          │   Google Cloud Storage      │
          │   gs://<bucket>/data/       │
          └──────────────┬──────────────┘
                         │ bq load
          ┌──────────────▼──────────────┐
          │        BigQuery             │
          │   <dataset>.<table>         │
          └──────────────┬──────────────┘
                         │ SQL / REST API
          ┌──────────────▼──────────────┐
          │        Cloud Run            │
          │   collection-market-tracker-frontend-admin        │
          └─────────────────────────────┘
```

**Why Git as source of truth?**
GCP billing outages (or account suspensions) take down GCS, BigQuery, and Cloud Run simultaneously. Because all data is committed here, the frontend admin can fall back to reading directly from raw GitHub URLs with zero GCP dependency.

## Repository layout

```
.
├── config.yaml              # GCP project, bucket, dataset, table names
├── data/
│   └── items.jsonl          # Newline-delimited JSON data files (one per BQ table)
├── schema/
│   └── items.json           # BigQuery table schemas
├── scripts/
│   ├── sync_to_gcs.sh       # Upload data/ to GCS
│   └── load_to_bq.sh        # Load from GCS into BigQuery
└── .github/
    └── workflows/
        └── sync.yml         # CI: run both scripts on every push to main
```

## Setup

### 1. Fill in `config.yaml`

Replace the placeholder values:

```yaml
gcp:
  project_id: my-gcp-project

gcs:
  bucket: my-data-bucket
  data_prefix: data/

bigquery:
  dataset: my_dataset
  tables:
    - items
```

### 2. Configure GitHub secrets

| Secret | Value |
|---|---|
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | Workload Identity provider resource name |
| `GCP_SERVICE_ACCOUNT` | Service account email with Storage and BigQuery permissions |

See [google-github-actions/auth](https://github.com/google-github-actions/auth) for how to set up keyless authentication via Workload Identity Federation.

### 3. Add your data

- Add `data/<table>.jsonl` files (one JSON object per line).
- Add a matching `schema/<table>.json` BigQuery schema file.
- Register the table name under `bigquery.tables` in `config.yaml`.

### 4. Push to `main`

The GitHub Actions workflow triggers automatically and syncs everything to GCS and BigQuery.

## Running scripts locally

```bash
# Authenticate first
gcloud auth application-default login

pip install pyyaml

bash scripts/sync_to_gcs.sh
bash scripts/load_to_bq.sh
```

## License

GPL-3.0 — see [LICENSE](LICENSE).
