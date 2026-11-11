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

**The watch is now forbidden from clicking at all.** Verified 11 Nov 2026 end to end: on a
fresh page with no click, the vendored document fetched straight from this repo resolved a
board. If it stops working — `getBreak` returns null, or errors on a break id you know
resolves — the run stops and reports. Re-lifting is a job for an attended session, because
the only way to re-lift is the click, and that is the step that opened the wallet once.

**Note:** `pull_boards.js` in the private repo still opens with the click instructions and
does not point here. Its click section is superseded. A run that reads only that file and
follows it will click for no reason.

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
- **Settled 11 Nov 2026: the 40% NFL-team rule is gone.** Whatnot classifies the stream,
  not the board: `LiveStream.livestreamCategories -> LivestreamTagNode` carries
  `label: "Football Cards"` / `deeplink: "tag/football"`, id `TGl2ZXN0cmVhbVRhZ05vZGU6OTYw`,
  and on 11 Nov all 21 streams in the feed carried exactly that one category. A break id
  harvested from the football feed is football; no content test is needed.
  The `Break` node has `format` and `spotType` but no sport field, and introspection is
  disabled (HTTP 400), so the category must be read off the stream at scrape time — which is
  another reason to capture the livestream UUID while you have it.
  Content survives only as a veto for boards a seller lists in the football category that are
  plainly something else (6336476, Funkos of wrestlers and actors). The four boards the old
  rule wrongly dropped — 6311737, 6338896, 6309608, 6338407 — were re-admitted.
- **The watch does no valuation.** It collects; Dennis evaluates on demand. The reason is
  that the bid board in the private repo is **2026 Inception hobby only** (EUR 450/box), and
  Inception is about three times the per-box economics of the products that fill most of the
  feed: sold-out hobby boards take a median EUR 579/box for Inception against 174 for
  Flagship and 178 for Heritage. Swept across Flagship and Heritage asks it marks nearly
  every spot a buy. Any on-demand check must filter to Inception, `PICK_YOUR`, `TEAM` spot
  type, with a known box count — or wait for per-product bid boards, which do not exist.
- `boxes` lands blank on rows the watch adds: it is parsed from the title by the private
  repo's build chain, which the watch does not run. Box counts for the 11 Nov alert were
  read from titles by hand. Worth wiring that parser into the watch.
- When evaluating on demand, remember auction boards make the ask test vacuous: every open
  spot on a EUR 1-start board is "below fair". Posted boards give takeable buys; auction
  boards give pre-bid caps, which is a different decision. On 11 Nov all five open Inception
  1-box auctions were EUR 1 across every team, and all four posted Inception boards were
  140–158% of box cost and failed the 115% filter — so the honest answer was no takeable buy
  at all, and the five auctions filled within the hour anyway.

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
