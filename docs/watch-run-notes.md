# Hourly watch — operational notes

What each run learns that the next run should not have to rediscover. The pricing model and
the method write-up live in the private repo; this file is only how the extraction behaves
in practice. **Nothing operational is kept in browser storage** — if a run needs state
between runs, it belongs in this repo.

## The Buy Spots click is avoidable

The browser recipe in the private repo (`scripts/pull_boards.js`) says to click **Buy
Spots** once per session to get the `QueryBreak` document into Apollo's `queryManager`.
That click is the one step that goes near payment UI, and it turns out not to be necessary.

`d.loc` is absent in this build, so `QueryBreak` is a plain GraphQL AST: it survives a JSON
round-trip and Apollo accepts the reconstructed object as a query. The document is vendored
here at **`scripts/querybreak.json`** (12,432 bytes, sha256 `11c8498723b9a2bc…`), lifted
11 Nov 2026 and verified against 274 live queries in that run.

```js
// DOC = the contents of scripts/querybreak.json
const D = JSON.parse(DOC);
await window.__APOLLO_CLIENT__.query({
  query: D, fetchPolicy: 'no-cache',
  variables: { id: String(id), first: 60, after,
    getPaginatedSpots: true, getPaginatedSpotOptions: false,
    getAllSpots: false, getAllSpotOptions: false,
    filters: [{ field: 'SOLD_STATUS', values: ['SOLD'] }] }});   // or 'UNSOLD'
```

A run still needs a live signed-in browser — Cloudflare and the ~2-minute access token are
unchanged — but it no longer needs to click anything, and it can navigate freely, because
the document no longer dies with the page.

Re-lift it with the original recipe only if Whatnot changes the query shape. Symptom:
`getBreak` returns null, or errors, on a break id you know resolves.

**Note:** `pull_boards.js` in the private repo still opens with the click instructions and
does not point here. A run that reads only that file will still click.

## Finding break ids

- The category feed (`Football Cards`, `/?feedId=TABBED_CATEGORY_FEED_V2:…`) holds only
  `LiveStream` nodes. **No break ids** — the LiveStream node has no break field. Scrolling
  stops loading at about 21 streams.
- Break ids appear in the Apollo cache as `Break:<id>` only after opening `/live/<uuid>`.
  Roughly half the football streams carry any; singles shows carry none, and some break
  shows return none until the shop panel is opened, which is not worth a click.
- Workable pattern: navigate, read the cache, accumulate, 5–6 streams per batched call.

## Cutting the data back out of the browser

`get_page_text` tops out around 25 KB per read and strips newlines, so records have to be
self-delimiting (`S|<id>|…`, split on `/(?=S\|)/`).

Pulling every active board costs roughly 220 KB of transcription. Hash first instead: build
a canonical per-board hash in the browser (rows sorted, `price.toFixed(2)`, `UNSOLD`→`OPEN`,
SHA-1 via `crypto.subtle`), compare it against the same hash computed from
`spots/<break_id>.csv`, and transcribe only the boards that differ. On 11 Nov that was 27
of 137 boards — about a 70% saving, and it doubles as the list of what actually moved.

The **metadata** still has to come back for every board: a board can go `breaking` →
`completed` with its spot rows untouched, which is what 6250438 did.

## Known data quirks

- `wn_sold_spot_count` can go **down** between runs, and can read 0 on a board whose spot
  rows show sales (5151046 and 5518931 on 11 Nov). It is Whatnot's own counter, not a
  transcription error. Treat `soldSpotCount` **or** `filledBreakSpots` matching the SOLD row
  count as reconciled; anything else is a real mismatch and the board should be skipped.
- 6294475 returned 32 UNSOLD spots against `totalBreakSpots` 29 — failed the checksum on
  11 Nov and was not merged. Worth re-checking whether it clears.
- The 40% NFL-team rule drops football boards whose spots are numbered or divisional: RTB
  boards (`#1`, `#2`, …) and pick-your-division boards (`AFC North`). On 11 Nov it dropped
  4 genuine football boards (6311737, 6338896, 6309608, 6338407) and 1 correct non-football
  board (6336476, autographed Funkos). This ledger already contains such boards from the
  backfill — 6251997 scores 0% — so the rule is stricter than the data it inherited.
  **Unsettled.**
- The bid board in the private repo is **2026 Inception hobby only** (EUR 450/box). Pointed
  at Flagship or Heritage asks it flags everything as a buy. The alert has to filter to
  Inception, `PICK_YOUR`, `TEAM` spot type, with a known box count.
- `boxes` lands blank on rows the watch adds: it is parsed from the title by the private
  repo's build chain, which the watch does not run. Box counts for the 11 Nov alert were
  read from titles by hand. Worth wiring that parser into the watch.
- Auction boards make the ask test vacuous — every open spot on a EUR 1-start board is
  "below fair". Report posted boards as takeable buys and auction boards separately, as
  pre-bid caps.

## Livestream UUIDs

The Caveats in the README note that a `break_id` cannot be turned back into a
`/live/<uuid>` link. Captured at scrape time, these can:

| Break ids | Stream | Seller |
|---|---|---|
| 6293376, 6293377, 6293380, 6293381, 6293382 | `e1a6caba-b051-4fa6-8d6e-76a3f8e8336c` | derkiosk |
| 6311681, 6311737, 6311777 | `1085322d-978d-4a30-80d8-b1fc8e02959d` | |
| 6309608, 6309657, 6309673 | `d7868294-2a36-481d-910d-58c97f5a6cf5` | |
| 6345323 | `dc67200a-68aa-48a4-9a78-3dc1089853e9` | |
| 6333513 | `c3b421a4-7265-48a7-b56d-7388c8e0c8bb` | |
