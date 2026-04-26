# collection-market-tracker-data

Static JSON snapshots for the **Collection Market Tracker** — a BigQuery + Cloud Run + GCS system that tracks TCG market prices and listings.

The backend publishes JSON files here after every write and on a daily cron schedule. Frontends read from these files via GitHub Raw URLs as their primary data source, with GCS and the live API as fallbacks.

**Do not manually edit files under `data/`** — the backend owns all writes.

---

## Architecture

```
BigQuery (source of truth)
  └── collection-market-tracker-backend
        ├── On every write: datasync.TriggerSync()
        └── Daily cron: collection-showcase-data-sync Cloud Run job
              │
              ├──► gs://collection-showcase-data/data/*.json   (GCS)
              └──► data/*.json  (this repo, via GitHub API commit)

Frontends (admin + public):
  GitHub Raw (this repo) ► GCS ► Live API
```

## Related Repositories

| Repo | Role |
|------|------|
| [collection-market-tracker-backend](https://github.com/FutureGadgetCollections/collection-market-tracker-backend) | Writes to this repo after every mutation and on cron |
| [collection-admin](https://github.com/FutureGadgetCollections/collection-admin) | Admin UI — reads from this repo as primary data source |
| [collection-showcase-frontend](https://github.com/FutureGadgetCollections/collection-showcase-frontend) | Public frontend — also reads from this repo |

## Data Files

All files are JSON arrays. Published to `data/` by the backend:

| File | BigQuery Table | Composite Key |
|------|---------------|---------------|
| `data/sealed-products.json` | `catalog.sealed_products` | `(game, set_code, product_type)` |
| `data/single-cards.json` | `catalog.single_cards` | `(game, set_code, card_number)` |
| `data/set-pull-rates.json` | `catalog.set_pull_rates` | `(set_code, rarity)` |

## GCP Infrastructure

| Resource | Details |
|----------|---------|
| GCP project | `future-gadget-labs-483502` |
| GCS bucket | `collection-showcase-data` — parallel copy of all data files |
| BigQuery | Project `future-gadget-labs-483502`, dataset `catalog` |
| Cloud Run service | `collection-market-tracker` — `us-central1` |
| Cloud Run job (cron) | `collection-showcase-data-sync` — `us-central1`, runs daily |

## Manually Triggering a Sync

To force a refresh, use the backend's sync endpoint (requires Firebase auth):

```
POST /sync/sealed-products
POST /sync/single-cards
POST /sync/set-pull-rates
```

Or use the **Sync** button in the admin frontend UI.

## License

GPL-3.0 — see [LICENSE](LICENSE).
