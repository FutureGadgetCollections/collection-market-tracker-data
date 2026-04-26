# collection-market-tracker-data

## Project Overview

Static JSON data layer for the **Collection Market Tracker**. This repo holds JSON snapshots of BigQuery tables, committed automatically by the backend after every write and on a daily cron schedule. Frontends read from it directly via GitHub Raw URLs as their primary data source.

**Do not manually edit files under `data/`** — the backend owns all writes here.

## Multi-Repo Setup

All repos are siblings under the same parent directory. Run `setup.sh` from `collection-admin` to clone any missing sibling repos.

## All Repositories

| Repo | GitHub | Local Path | Purpose |
|------|--------|-----------|---------|
| Data files (this repo) | `FutureGadgetCollections/collection-market-tracker-data` | `../collection-market-tracker-data` | JSON snapshots published by backend |
| Backend | `FutureGadgetCollections/collection-market-tracker-backend` | `../collection-market-tracker-backend` | Owns writes to this repo + GCS |
| Frontend admin | `FutureGadgetCollections/collection-admin` | `../collection-admin` | Admin UI — reads from this repo as primary source |
| Showcase frontend (public) | `FutureGadgetCollections/collection-showcase-frontend` | `../collection-showcase-frontend` | Public Hugo site — also reads from here |

## GCP Infrastructure

| Resource | Details |
|----------|---------|
| GCP Project | `future-gadget-labs-483502` |
e| GCS bucket | `collection-tracker-data` — parallel copy of data files |
| BigQuery | Project `future-gadget-labs-483502`, dataset `catalog` — source of truth |
| Cloud Run service | `collection-market-tracker` — writes to this repo after every mutation |
| Cloud Run job | `collection-showcase-data-sync` — daily cron that refreshes data files |

## Data Flow

```
BigQuery (source of truth)
  └── Backend API (on mutation) or Cron (daily)
        ├──► gs://collection-tracker-data/data/<resource>.json  (GCS)
        └──► data/<resource>.json (this repo, via GitHub API commit)

Frontends read priority: GitHub Raw (this repo) ► GCS ► Live API
```

## Data Files

All files are JSON arrays (`[]` when empty). The backend publishes to `data/` and `schema/`:

| File | BQ Table | Composite Key |
|------|----------|---------------|
| `data/sealed-products.json` | `catalog.sealed_products` | `(game, set_code, product_type)` |
| `data/single-cards.json` | `catalog.single_cards` | `(game, set_code, card_number)` |
| `data/set-pull-rates.json` | `catalog.set_pull_rates` | `(set_code, rarity)` |
| `schema/sealed-products.json` | — | BigQuery field definitions for sealed products |
| `schema/single-cards.json` | — | BigQuery field definitions for single cards |
| `schema/set-pull-rates.json` | — | BigQuery field definitions for set pull rates |

## Why Git as a Fallback?

GCP billing outages (or account suspensions) take down GCS, BigQuery, and Cloud Run simultaneously. Because all data is committed here, frontends can fall back to GitHub Raw URLs with zero GCP dependency.

## Manually Triggering a Sync

To force a refresh without waiting for the cron job, use the backend's sync endpoint (requires Firebase auth):

```
POST /sync/sealed-products
POST /sync/single-cards
POST /sync/set-pull-rates
```

Or use the **Sync** button in the admin frontend UI.
