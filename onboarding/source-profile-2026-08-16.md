# Oakville source profile — probes of 2026-08-16

Locked probes per `04-city-onboarding.md`, run against the tracker DB
(`ontario-address-changes/data/oakville/oakville.db`) and the town's own ArcGIS
service. Everything in `config.toml` traces back to a number here.

Re-run these before trusting them; the source is live and the tracker is still
pulling.

## Snapshot history

26 snapshots, 2026-06-10 → 2026-08-12. Snapshots 2–19 are `skipped=1`
(unchanged content), so the tracker was deduplicating correctly.

| snapshot | date | rows |
|---|---|---|
| 1 | 2026-06-10 | 71,051 |
| 20 | 2026-07-03 | 71,055 |
| 21 | 2026-07-08 | 71,056 |
| 22 | 2026-07-16 | 71,052 |
| 23 | 2026-08-01 | 71,052 |
| 24 | 2026-08-05 | 71,048 |
| 25 | 2026-08-06 | 71,049 |
| **26** | **2026-08-12** | **71,049** |

A live, actively maintained layer that churns by single digits per pull — no
freshness caveat, unlike the stalled trackers in Waterloo (stopped 2026-06-27)
or Lambton (2026-06-28).

## Field population (snapshot 26, 71,049 active rows)

| field | populated | note |
|---|---|---|
| `number` | 71,048 | one null — see "The null row" below |
| `street` | 71,049 | SNAME, 100% |
| `unit` | 5,896 | 8.3% |
| `full` | 71,048 | ADDRESS |
| lat/lon | 71,049 | 100% |

Props key sets, by frequency:

```
63,044  ADDRESS, LABEL, SNAME, STREET_NAME, STREET_NUM, STREET_TYPE
 5,503  ... + UNIT
 1,995  ... + STREET_DIR
   393  ... + STREET_DIR + UNIT
    85  ADDRESS, LABEL, SNAME, STREET_NAME, STREET_NUM        (no STREET_TYPE)
    24  ... + SUFFIX                                          (14 with DIR, 10 without)
     4  ADDRESS, SNAME, STREET_NAME, STREET_NUM, STREET_TYPE  (no LABEL)
```

Seven columns exist in the source schema but are empty on every row of every
snapshot and never reach the tracker: `SUITE`, `TOP_ALIAS`, `ADDR_ALIAS`,
`POSTAL`, `CITY`, `PROV`, `STREET_TYPE_PREFIX`, `STREET_DIR_PREFIX`. The
`SUITE`/`UNIT` question was settled in `ontario-address-changes/datasets/oakville.toml`
(2026-08-10): `SUITE` is an empty schema slot, not a second unit column, and
remapping `unit` to it would blank 5,896 real values.

## Finding 1 — street names are UPPERCASE, and the normalizer cannot fix them

**This is the blocker.** 1,646 of 1,647 distinct street names are all-caps.
The single exception is `Maple Grove Dr` — one row of source inconsistency.

`t2.conflate.expand_street_name()` rewrites only the trailing direction token
and the suffix token before it, preserving earlier tokens verbatim (by design —
it exists to turn "Foo Ave W" into "Foo Avenue West"). Run against real Oakville
values it produces:

```
'CHANCERY LANE E'    -> 'CHANCERY LANE East'
'CAMEO ST'           -> 'CAMEO Street'
'AVON CRES'          -> 'AVON Crescent'
'TRAFALGAR RD'       -> 'TRAFALGAR Road'
'SIXTH LINE'         -> 'SIXTH LINE'
'WINDING WOODS DR'   -> 'WINDING WOODS Drive'
```

Every one of those is wrong for `addr:street`. Note the second failure mode:
type tokens the source already spells out are not in the short→long expansion
table, so they pass through still uppercase — `LANE` (86), `TRAIL` (49),
`GATE` (40), `COMMON` (35), `WAY` (62), `LINE`.

Street type token distribution (last token, skipping a trailing direction):

```
DR 365 · RD 225 · CRES 214 · CRT 141 · ST 108 · AVE 97 · LANE 86 · WAY 62
PL 60 · TRAIL 49 · GATE 40 · BLVD 39 · COMMON 35 · CIR 29 · GDNS 23
```

### Edge cases any title-caser must handle

Python's `str.title()` gets four of these wrong, so do not reach for it:

| class | examples | `.title()` gives | correct |
|---|---|---|---|
| apostrophe-S | `STILL'S LANE`, `MUNN'S AVE`, `KING'S COLLEGE DR`, `WATER'S EDGE DR` | `Still'S Lane` | `Still's Lane` |
| O'-prefix | `O'DONOGHUE AVE` | `O'Donoghue Ave` ✓ | `O'Donoghue Avenue` |
| Mc-prefix | `MCCRANEY ST E`, `MCDOWELL AVE`, `MCPHERSON RD`, `MCWILLIAMS CRES`, `HAZEL MCCLEARY DR`, `JOHN MCKAY BLVD` | `Mccraney` | `McCraney` |
| Mac-prefix | `MACDONALD RD`, `MACLENNAN DR`, `DONALD MACKAY ST` | `Macdonald` | `MacDonald` (judgment — check OSM) |
| abbreviated saint | `ST. ANN'S CRT` | `St. Ann'S Crt` | `St. Ann's Court` (judgment) |
| leading article | `THE LINKS DR`, `THE GREENERY`, `THE PROMENADE`, `THE STABLES`, `THE CANADIAN RD` | fine | fine |
| numeric | `REGIONAL ROAD 25`, `HIGHWAY 407 (WB)` | fine | `(WB)` is a ramp — probably not addressable |

No hyphenated names exist in the dataset (checked). `expand_street_name()`
already has a `_glue_mc_prefix` helper, but it only reassembles `Mc Caul` →
`McCaul` from *separated* tokens and explicitly does not re-case the surname —
so it does not solve `MCCRANEY`.

**The authority on the right form is OSM itself**, not a general rule: Oakville
already has thousands of mapped street names, many of them touched by
TronnaLegacy. Diff the normalized source names against OSM's `name` values on
nearby ways before settling the table — that is what `t2.streets` /
`/streets` in the UI is for.

## Finding 2 — the tracker's `number` column drops housenumber suffixes

24 rows carry a `SUFFIX` prop. Their true addresses are `335A LAKESHORE RD E`,
`339A LAKESHORE RD E`, `338A CHURCH ST`, `340A CHURCH ST`, `324A FELAN AVE`,
`276A KERR ST`, … but the tracker stores:

```
number = "335"   street = "LAKESHORE RD E"   full = "335A LAKESHORE RD E"
props: STREET_NUM = "335", SUFFIX = "A", LABEL = "335"
```

Uploading `addr:housenumber=335` there is both wrong and a collision with the
genuine 335 next door. `number GLOB '*[A-Za-z]*'` returns 0 rows, confirming
the column is suffix-free throughout.

`LABEL` is not the fix: it equals `STREET_NUM` on plain rows but holds the
**unit** on all 5,896 unit rows (`LABEL = "33"` for `1135 MCCRANEY ST E UNIT 33`).

Only `ADDRESS` carries the combined form, which forces
`number_from = "full"` — and therefore forces `full_from = "full"`, since the
engine rejects `number_from = "full"` alongside `full_from = "number+street"`
as circular (`t2/config.py:147`). That cascade is why this checkout's
`[source_fields]` looks different from Guelph's.

## Finding 3 — ADDRESS embeds the unit

```
1135 MCCRANEY ST E UNIT 33
1450 GLEN ABBEY GATE UNIT 1
2400 MUNN'S AVE UNIT A
```

The Guelph trap (`08-survey-results-2026-08-12.md`): any `DISTINCT full` count
is inflated by unit rows. Guelph avoided it with `full_from = "number+street"`;
Oakville cannot, per Finding 2. Collapse-to-civic keys on
(number, street, municipality) rather than on `full`, so candidates should be
unaffected — **verify on the first run** rather than trusting that.

## Finding 4 — the null row

One row, identity `syn:42a4d668…`, has no `STREET_NUM`, no `ADDRESS`, no
`LABEL`, and no unit — only `SNAME = "CHARLES BRAY CRES"`. With
`number_from = "full"` it projects a NULL housenumber. Expect it to skip or
flag; confirm it does not silently become an address node.

## Civic collapse

```
71,049 active rows  ->  65,135 distinct (number, street)
```

5,914 rows collapse away, consistent with the 5,896 unit rows plus a few
intra-source duplicates. Worth a look during review: civic groups holding more
than one unit-less row.

## Extent

```
lat  43.37207 .. 43.5237
lon  -79.8097 .. -79.62587
```

## Tile fabric — Wards, 100.00% coverage

Probed the town's self-hosted ArcGIS at
`maps.oakville.ca/oakgis/rest/services`. The `OPEN` folder holds nothing
useful (two unrelated services); the fabric is in the **same MapServer as the
address points**, `ArcGISOnline/AGO_Oakville_Misc_Layers`:

- **layer 21 `Wards`** — 7 polygons, `maxRecordCount` 1000, fields include
  `WARD` ("1".."7") and `FULL_NAME` (identical values).
- layer 16 `Municipal Boundary` — 1 polygon, the future
  `10-boundary-clipping.md` input. Noted so it is not re-discovered.
- No neighbourhood / community / planning-district layer exists. Wards are the
  only option, and they are enough.

Point test, 71,049 addresses against the 7 ward polygons (shapely, `covers`):

```
inside   71,048   =>  100.00%
orphans       1   (the null row of Finding 4)

ward 1   9,444      ward 5  10,241
ward 2   8,966      ward 6  11,092
ward 3   8,292      ward 7   8,791
ward 4  14,222
```

Best coverage in the portfolio (Hamilton 99.99%, Guelph 99.47%, Quinte West's
winning ward layer 99.87%). Raw output: `ward-coverage-2026-08-16.json`.

## Not probed yet

- **The OSM-side gap.** The 2026-08-12 portfolio survey said 49,322 of 65,134
  missing (75.7%), but TronnaLegacy has been adding addresses since 2026-08-08.
  That number is stale by construction — **re-measure it**, do not quote it.
  It is item 8 of `IMPORT_CHECKLIST.md`.
- Gap distribution (500 m cells), provenance/last-touch editor tally: both need
  the Geofabrik extract downloaded first.
