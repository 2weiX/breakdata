# breakdata

Whatnot **football** break observations: what sold, for how much, on which board.

Football only. Boards from other categories (NBA, Magic, Pokémon, Star Trek) are excluded
even where they appear in the owner's order history. Classification is by spot content —
a board counts as football when at least 40% of its spots start with an NFL team name —
not by title, which mislabels boards like "Team Rest 3 Caser Flagship 2026".

Append-only. Rewritten in full on each observation; git carries the history.

## Layout

| Path | What |
|---|---|
| `breaks.csv` | one row per break ever seen |
| `spots/<break_id>.csv` | every spot on that break |

## breaks.csv

Columns prefixed `wn_` are **delivered by Whatnot**. Everything else is derived locally and
may be wrong; the `wn_` columns are the source of truth.

`break_id, wn_format, wn_status, wn_spot_type, wn_transaction_type, wn_sold_spot_count,
wn_filled_break_spots, wn_total_break_spots, product, seller, title, boxes, box_kind,
take_eur, asks_eur, first_seen, spots_file, sha1`

| Field | Values | Notes |
|---|---|---|
| `break_id` | integer | Whatnot's own id. Resolves via `getBreak` from any seller, forever — all 312 re-resolved with zero misses, including October boards. |
| `wn_format` | `PICK_YOUR` 222, `RANDOM` 89, `MYSTERY` 1 | `RANDOM` spots are lottery tickets, not team prices — exclude from per-team statistics |
| `wn_spot_type` | `TEAM` 237, `CUSTOM` 70, `DIVISION` 5 | `CUSTOM` is player/college/mixed; needs summing to team level before it contributes |
| `wn_transaction_type` | `BUY_IT_NOW` 221, `AUCTION` 91 | use this; do not infer mechanic from price spread |
| `wn_status` | `completed` 194, `breaking` 96, `filling` 21, `cancelled` 1 | per-unit statistics should require a sold-out board |
| `wn_sold_spot_count` vs `wn_filled_break_spots` | differ on 35 breaks | filled includes spots that are not sold; use `sold` for market stats |
| `boxes` | present on 234 of 312 | parsed from the title. **Weakest column here** — it feeds every per-box number |
| `first_seen` | date | when this ledger first saw the board, *not* when Whatnot created it |
| `sha1` | 12 hex | of the spot file; unchanged hash means unchanged board, skip without parsing |

## spots/<break_id>.csv

`spot, price, state, buyer`

`state` is `SOLD` or `OPEN`; an open row's price is the current ask and `buyer` is empty.
Prices are EUR. Rows are a point-in-time snapshot — a board that moved since its last pull
is stale and nothing in the file says so.

## Caveats

- The livestream UUID is not recorded, so a `break_id` resolves through the API but cannot
  be turned back into a `whatnot.com/live/<uuid>` link.
- Buyer usernames are as they appear publicly on Whatnot during each live show.
- No `reference/` or `bids/` yet; those are proposed, not built.
- One board (`6252372`, a 87-spot pick-your-player board) holds 63 of its 87 sold spots;
  the rest were not transcribed.

Model, scripts and method live in a separate private repo.
