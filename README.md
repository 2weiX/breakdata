# breakdata

Whatnot **football** break observations: what sold, for how much, on which board.

Football only. Boards from other categories (NBA, Magic, Pokémon, Star Trek) are excluded
even where they appear in the owner's order history.

**Classification is Whatnot's own, not ours.** A livestream carries
`livestreamCategories -> LivestreamTagNode`, and the football one is
`label: "Football Cards"`, `deeplink: "tag/football"`, id `TGl2ZXN0cmVhbVRhZ05vZGU6OTYw`.
A break id harvested from that category feed is football, full stop. The `Break` node itself
carries `format` and `spotType` but no sport, so the category has to come from the stream the
board was found in — capture it at scrape time.

Spot content is only a **veto**, for the case Whatnot's category cannot catch: a seller runs a
football show and slips in a break of something else. A board whose spots plainly name another
category's subjects is dropped by hand, with the reason recorded. Content is not the gate.

Earlier revisions of this file defined football as "at least 40% of spots start with an NFL
team name". That test was written for one cleanup of the order-history backfill, where every
board had team-named spots, and it was wrong as a standing rule: it cannot see a
random-team board that numbers its spots `#1`, `#2`, or a pick-your-division board that names
`AFC North`. It dropped four genuine football boards on 11 Nov 2026 before being removed.

Append-only. Rewritten in full on each observation; git carries the history.

## Layout

| Path | What |
|---|---|
| `breaks.csv` | one row per break ever seen |
| `spots/<break_id>.csv` | every spot on that break |
| `scripts/querybreak.json` | the lifted `QueryBreak` document, so a run needs no Buy Spots click |
| `docs/watch-run-notes.md` | how the extraction behaves in practice: quirks, costs, livestream UUIDs |

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

- The livestream UUID is not a column here, so a `break_id` resolves through the API but
  cannot be turned back into a `whatnot.com/live/<uuid>` link. The ones captured since the
  watch started are listed in `docs/watch-run-notes.md`.
- Buyer usernames are as they appear publicly on Whatnot during each live show.
- No `reference/` or `bids/` yet; those are proposed, not built.
- `6336476` (2x Gold Rush Autographed Funkos Series 2) was listed in the Football Cards
  category but its spots are wrestlers, baseball players and actors. Held out by the
  content veto, not by Whatnot's category.
- `6252372` (87-spot pick-your-player) held only 63 of its 87 sold spots until the
  11 Nov 2026 watch re-pulled it; it is now complete.

The pricing model and the method write-up live in a separate private repo. What lives here
is the data plus whatever the harness needs to reproduce the pull: `scripts/` and `docs/`.
Read `docs/watch-run-notes.md` before running the extraction — it says how to skip the
Buy Spots click, which is the one step that goes near payment UI.
