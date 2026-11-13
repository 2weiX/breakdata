# breakdata

Whatnot **football** break observations: what sold, for how much, on which board.

Football only. Boards from other categories (NBA, Magic, Pokémon, Star Trek) are excluded
even where they appear in the owner's order history.

Classification is by the Whatnot category the board was found in, recorded at scrape time;
spot content is a hand-applied veto for the case where a seller runs a football show and
slips in a break of something else. Title is not a test — it mislabels boards like
"Team Rest 3 Caser Flagship 2026". Spot content alone is not a test either: it cannot see a
random-team board whose spots are numbered `#1`, `#2`, or a pick-your-division board whose
spots read `AFC North`. The 321 boards here were each checked by hand; boards added later
inherit the category they were harvested from.

Append-only. Rewritten in full on each observation; git carries the history.

## Layout

| Path | What |
|---|---|
| `breaks.csv` | one row per break ever seen |
| `spots/<break_id>.csv` | every spot on that break |

## breaks.csv

Columns prefixed `wn_` are **delivered by Whatnot**. Everything else is derived locally and
may be wrong; the `wn_` columns are the source of truth.

Counts in the table below are as of 13 Nov 2026: **382 boards, €262,621.61 taken**. They go stale on every pull. Recompute; do not trust.

`break_id, wn_format, wn_status, wn_spot_type, wn_transaction_type, wn_sold_spot_count,
wn_filled_break_spots, wn_total_break_spots, product, seller, title, boxes, box_kind,
take_eur, asks_eur, first_seen, spots_file, sha1, stream_uuid, description, stream_url`

| Field | Values | Notes |
|---|---|---|
| `break_id` | integer | Whatnot's own id. Resolves via `getBreak` from any seller, forever — every board here has re-resolved with zero misses, including August boards. |
| `wn_format` | `PICK_YOUR` 234, `RANDOM` 87 | `RANDOM` spots are lottery tickets, not team prices — exclude from per-team statistics |
| `wn_spot_type` | `TEAM` 251, `CUSTOM` 64, `DIVISION` 6 | `CUSTOM` is player/college/mixed; needs summing to team level before it contributes |
| `wn_transaction_type` | `BUY_IT_NOW` 230, `AUCTION` 91 | use this; do not infer mechanic from price spread |
| `wn_status` | `completed` 186, `breaking` 95, `filling` 39, `cancelled` 1 | per-unit statistics should require a sold-out board. `breaking` is sticky: boards sit in it long after the show ends, so it does not mean live. |
| `wn_sold_spot_count` vs `wn_filled_break_spots` | differ on 37 breaks | filled includes spots that are not sold; use `sold` for market stats |
| `boxes` | present on 233 of 321 | parsed from the title by the private repo's build chain. **Weakest column here** — it feeds every per-box number |
| `first_seen` | date | when this ledger first saw the board, *not* when Whatnot created it |
| `sha1` | 12 hex | of the spot file; unchanged hash means unchanged board, skip without parsing |
| `stream_uuid` | uuid | the show the board was found in. Captured since 13 Nov 2026, so blank on older boards |
| `stream_url` | url | `whatnot.com/live/<stream_uuid>`; blank when the uuid is |
| `description` | text | the seller's own blurb beside the title. Often carries the box count and product when the title does not. Newlines collapsed to spaces |

## spots/<break_id>.csv

`spot, price, state, buyer`

`state` is `SOLD` or `OPEN`; an open row's price is the current ask and `buyer` is empty.
Prices are EUR. Rows are a point-in-time snapshot — a board that moved since its last pull
is stale and nothing in the file says so.

## Caveats

- Buyer usernames are as they appear publicly on Whatnot during each live show.
- No `reference/` or `bids/` yet; those are proposed, not built.

Model, scripts and method live in a separate private repo.
