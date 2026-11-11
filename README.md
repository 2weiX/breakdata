# breakdata

The wire between the browser-side Whatnot watch and the harness pricing engine.

The Whatnot session stays in the browser: `getBreak` requires a short-lived access token
(observed lifetime ~2 minutes) obtained from an HttpOnly refresh cookie that JavaScript
cannot read. The harness therefore never talks to Whatnot. It reads this repo instead.

## Layout

| Path | Owner | What |
|---|---|---|
| `breaks.csv` | browser watch | the index: one row per break ever seen |
| `spots/<break_id>.csv` | browser watch | every spot on that break: name, price, state, buyer |
| `reference/` | harness | odds, checklists, formats, card prices (not yet populated) |
| `bids/` | harness | pricing engine output (not yet populated) |

One owner per directory. Neither side writes the other's, so every pull is fast-forward
and no merge logic exists anywhere.

## breaks.csv

Columns prefixed `wn_` are **delivered by Whatnot**, not inferred here. Everything else is
derived locally and may be wrong; the `wn_` columns are the source of truth.

`break_id, wn_format, wn_status, wn_spot_type, wn_transaction_type, wn_sold_spot_count,
wn_filled_break_spots, wn_total_break_spots, product, seller, title, boxes, box_kind,
take_eur, asks_eur, first_seen, spots_file, sha1`

| Field | Values seen | Notes |
|---|---|---|
| `break_id` | integer | Whatnot's own id. Resolves via `getBreak` from any seller, forever — all 312 re-resolved with zero misses, including October boards. |
| `wn_format` | `PICK_YOUR` 222, `RANDOM` 89, `MYSTERY` 1 | replaces the locally-inferred RTB heuristic |
| `wn_spot_type` | `TEAM` 237, `CUSTOM` 70, `DIVISION` 5 | the claim unit, product-agnostic |
| `wn_transaction_type` | `BUY_IT_NOW` 221, `AUCTION` 91 | replaces the "min sold price <= 8" heuristic |
| `wn_status` | `completed` 194, `breaking` 96, `filling` 21, `cancelled` 1 | |
| `wn_sold_spot_count` vs `wn_filled_break_spots` | differ on 35 breaks | filled includes spots that are not sold; use `sold` for market stats |

**Not on the Break node: the livestream id.** A break id resolves through the API but cannot
be turned back into a `whatnot.com/live/<uuid>` URL. The watch must record the livestream
UUID from the page URL at scrape time; it cannot be recovered afterwards.

`sha1` is the first 12 hex of the spot file's SHA-1 — a changed board changes its hash,
so the harness can skip unchanged files without parsing them.

## spots/<break_id>.csv

`spot, price, state, buyer`

`state` is `SOLD` or `OPEN`. An open row's price is the current ask. `buyer` is empty on
open rows. Prices are EUR, already divided by 100.

## Rules

- **Append and overwrite, never delete.** A break's spot file is rewritten in full on each
  observation; `breaks.csv` is rewritten in full. Git carries the history.
- **Break ids are stable and sequential.** The API resolves any id from any seller,
  forever, so a break that has finished is still re-readable.
- **Checksum before writing.** Transcribed totals must equal the API's own
  `soldSpotCount` and take, or the run writes nothing.
- No tokens, no cookies, no personal data. Buyer usernames are public on the board.

Review package, model outputs and build scripts: https://github.com/2weiX/breakreview
