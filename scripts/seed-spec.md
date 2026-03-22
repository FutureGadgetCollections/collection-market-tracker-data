# Seed Script Specification

Populates the backend's `catalog` dataset by reading the 6 source JSON files in `data/` and
POSTing each record to the backend REST API. Idempotent: a 409 response means the record already
exists and is silently skipped.

---

## Source files

Two files per game — one for set-level data (sealed products + pull rates), one for card-level data.

| File | Feeds tables | Game |
|---|---|---|
| `data/pokemon-sets.json` | `sealed_products`, `set_pull_rates` | pokemon |
| `data/pokemon-cards.json` | `single_cards` | pokemon |
| `data/one-piece-sets.json` | `sealed_products`, `set_pull_rates` | one-piece |
| `data/one-piece-cards.json` | `single_cards` | one-piece |
| `data/riftbound-sets.json` | `sealed_products` | riftbound |
| `data/riftbound-cards.json` | `single_cards` | riftbound |

---

## File structures

### Pokemon and One Piece (outer key present)

```json
{
  "sealed_products": [ { ... }, ... ],
  "pull_rates":      [ { ... }, ... ]
}
```

```json
{
  "cards": [ { ... }, ... ]
}
```

The script reads the array under the named key.

### Riftbound (no outer key — flat array at root)

```json
[ { ... }, ... ]
```

The script reads the root array directly. Both `riftbound-sets.json` and `riftbound-cards.json`
use this flat structure. Riftbound has no pull rate data; `riftbound-sets.json` only feeds
`sealed_products`.

---

## Normalization

Before posting any record, apply these transforms:

1. **Empty string → null**: any field whose JSON value is `""` must be replaced with `null`
   (i.e. the field is omitted from the POST body or set to `null`). This affects optional string
   fields such as `pricecharting_url`, `tcgplayer_id`, `standard_legal_until`.

2. **One Piece pull rate missing fields**: `unique_card_count`, `individual_card_pull_rate`, and
   `master_difficulty_score` will be absent in One Piece pull rate objects. Treat missing fields
   as `null`; do not fabricate values.

---

## Endpoint mapping

| Table | POST endpoint |
|---|---|
| `sealed_products` | `POST /sealed-products` |
| `single_cards` | `POST /single-cards` |
| `set_pull_rates` | `POST /set-pull-rates` |

See the backend README for the full request body schema for each endpoint.

---

## 409 handling

A `409 Conflict` means the composite key already exists in BigQuery. **Treat it as a no-op** —
log a skip message and continue to the next record. Do not retry and do not treat it as an error.

Any other non-2xx response is a hard error: log the status, body, and offending record, then
abort.

---

## Ordering

Seed in this order to satisfy any potential foreign-key intent and keep logs readable:

1. `sealed_products` (all three games)
2. `set_pull_rates` (pokemon + one-piece; riftbound skipped)
3. `single_cards` (all three games)

Within each table, process games in order: pokemon → one-piece → riftbound.

---

## Configuration

| Variable | Description |
|---|---|
| `SEED_BASE_URL` | Backend base URL, e.g. `http://localhost:8080` or the Cloud Run URL |
| `SEED_API_TOKEN` | Firebase ID token (omit when running against a local dev server with no auth) |
| `SEED_DRY_RUN` | If set to `true`, log what would be posted but make no HTTP calls |

---

## Expected output (per record)

```
[SKIP]  sealed_products  pokemon  sv01  booster-box       (409 already exists)
[OK]    sealed_products  pokemon  sv02  booster-box       (201)
[SKIP]  set_pull_rates   pokemon  sv01  double_rare       (409 already exists)
[OK]    set_pull_rates   pokemon  sv02  double_rare       (201)
[ERROR] single_cards     one-piece  op01  001              (500 — <body>)
```

---

## Implementation notes

- The script can be written in Go (`cmd/seed/main.go`) to reuse the existing module, or as a
  standalone shell/Python script. Go is preferred for consistency.
- The dry-run mode should print the full POST body for each record to make it easy to audit the
  normalization logic before a live run.
- Concurrency is optional; a simple sequential loop is fine for a one-time seed.
