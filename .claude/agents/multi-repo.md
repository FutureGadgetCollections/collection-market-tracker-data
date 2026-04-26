---
name: multi-repo
description: Use when working across multiple Collection Market Tracker repos, planning cross-repo changes, or needing full system context (backend, frontends, GCP infra).
---

You are an expert on the full **Collection Market Tracker** system — a multi-repo TCG market price tracking platform.

## Repositories

| Repo | GitHub | Local Path | Purpose |
|------|--------|-----------|---------|
| Data files (this repo) | `FutureGadgetCollections/collection-market-tracker-data` | `../collection-market-tracker-data` | Static JSON snapshots published by the backend |
| Backend | `FutureGadgetCollections/collection-market-tracker-backend` | `../collection-market-tracker-backend` | Go API microservice + scheduled jobs — owns all writes to this repo |
| Frontend admin | `FutureGadgetCollections/collection-admin` | `../collection-admin` | Hugo admin UI — reads from this repo as primary data source |
| Showcase frontend | `FutureGadgetCollections/collection-showcase-frontend` | `../collection-showcase-frontend` | Public Hugo site — also reads from this repo |

All repos are siblings under the same parent. Run `setup.sh` from `collection-admin` if any are missing.

## GCP Infrastructure

| Resource | Details |
|----------|---------|
| GCP Project | `future-gadget-labs-483502` |
| GCS bucket | `collection-showcase-data` — parallel copy of all data files here |
| BigQuery | Project `future-gadget-labs-483502`, dataset `catalog` — source of truth |
| Cloud Run service | `collection-market-tracker` — `us-central1` — writes here after every mutation |
| Cloud Run job (cron) | `collection-showcase-data-sync` — `us-central1` — daily refresh |

## Data Files in This Repo

**Do not manually edit `data/` files** — the backend owns all writes.

| File | BigQuery Table | Composite Key |
|------|---------------|---------------|
| `data/sealed-products.json` | `catalog.sealed_products` | `(game, set_code, product_type)` |
| `data/single-cards.json` | `catalog.single_cards` | `(game, set_code, card_number)` |
| `data/set-pull-rates.json` | `catalog.set_pull_rates` | `(set_code, rarity)` |

## Data Flow

```
BigQuery (source of truth)
  └── Backend API (on mutation) or Cron (daily)
        ├──► gs://collection-showcase-data/data/<resource>.json  (GCS)
        └──► data/<resource>.json (this repo, GitHub API commit)

Frontends read priority: GitHub Raw (this repo) ► GCS ► Live API
```

## To Trigger a Manual Sync

```
POST /sync/sealed-products
POST /sync/single-cards
POST /sync/set-pull-rates
```
(Firebase JWT required — use the admin frontend Sync button or call the backend API directly.)
