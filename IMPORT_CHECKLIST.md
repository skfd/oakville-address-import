# Oakville import checklist

Written 2026-08-16 for **TronnaLegacy**. Work it top to bottom — the order is
load-bearing, and each item says why it exists so you can argue with it.

Nothing here is urgent. You are already adding Oakville addresses by hand and
that work is fine as it is; this checklist is about what has to be true before
a *tool* does it in bulk under your account.

## Who owns which gate

| gate | owner | status |
|---|---|---|
| **Licence** — ODbL compatibility of the Town of Oakville OGL, cleared with the OSM Licensing Working Group | **skfd** | in progress, not yours to chase |
| **Street-name casing** — the source is UPPERCASE and the engine cannot fix it yet | you (code) | **blocking, item 2** |
| **Import proposal** — plan, `imports@` announcement, dedicated account | you | items 7–9 |
| **Is the gap even worth importing** | you | item 1 |

Two hard rules that override everything below:

1. **Nothing goes to the production OSM server until both the licence gate and
   the import-proposal gate are closed.** Dev-server runs are unlimited and
   invisible — use them freely.
2. **`[export] import_plan` in `config.toml` is deliberately empty, and the
   engine refuses to build changeset tags while it is.** That blocks both the
   tool's own upload *and* the JOSM `.osm` export — `changeset_tags()` raises
   either way (`t2/osm_export.py:66`), so there is no accidental side door.
   Fill it with your wiki plan URL when you have one; never fill it just to
   make the error go away.

---

## Phase 0 — Get it running

- [ ] **1.** Clone the three repos as siblings. The relative paths in
      `config.toml` assume this exact layout:

      Code/
        address-importer-friend/     # the engine (the tool itself)
        oakville-address-import/     # this repo
        ontario-address-changes/     # the source-data tracker

- [ ] **2.** Set up the engine per its own README: Python 3.11+, a venv,
      `pip install -e .` from `address-importer-friend/`.

- [ ] **3.** Register an OAuth2 app on the **dev** server
      (`master.apis.dev.openstreetmap.org`), redirect URI
      `http://127.0.0.1:5000/oauth/callback`. Copy `.env.dev.example` to
      `.env.dev` here and fill in the client id/secret, plus a random
      `FLASK_SECRET_KEY` and a `FERNET_KEY`
      (`python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`).
      Leave `.env.prod` alone for now — you cannot use it yet anyway.

- [ ] **4.** Get the Oakville tracker DB. It is
      `ontario-address-changes/data/oakville/oakville.db` (~32 MB), currently
      only on skfd's machine — **ask him for it**, along with how it gets
      refreshed. Everything downstream reads from it, so this is the one
      dependency you cannot work around.

- [ ] **5.** Download the OSM extract (~600 MB–1 GB, once, shared across every
      city):

      cd address-importer-friend
      python -m t2.osm_refresh

- [ ] **6.** Build the tile layer — this reads the 7 ward polygons and
      quadtree-splits them into ~250–750 address tiles:

      python -m t2.tiles_build

- [ ] **7.** Launch and confirm the dashboard, the map, and the DEV badge:

      python run.py --city-dir ../oakville-address-import

      Check the header says **DEV**. Get in the habit now — that badge is the
      only thing between a test and a real edit.

---

## Phase 1 — Measure the real gap (go/no-go)

- [ ] **8.** **Re-measure how many addresses are actually missing.** The
      portfolio survey of 2026-08-12 said 49,322 of 65,134 missing (75.7%), but
      you have been adding addresses since 2026-08-08, so that number is stale
      the moment it was written. Run a conflation over a few tiles and read the
      MISSING rate.

      This is the go/no-go. If your manual pace is already closing the gap, an
      import may not be the right instrument at all, and a smaller MapRoulette
      + QA workflow might serve Oakville better. Decide with a number in hand.

- [ ] **9.** **Look at the gap's shape, not just its size.** Concentrated in a
      few subdivisions (new construction you haven't reached) is a different
      problem from diffuse (leftovers, edge cases, disagreements). Bucket the
      missing addresses spatially before deciding.

- [ ] **10.** **Check what the source says about places you already mapped.**
      The interesting output of the first runs is not MISSING, it is
      MATCH_FAR and street-name disagreements: places where the source and your
      existing work differ. Those are worth reading one by one — they tell you
      whether the source is trustworthy at street level.

---

## Phase 2 — Fix the street casing (blocking)

Full evidence: `onboarding/source-profile-2026-08-16.md`, Finding 1.

The source publishes `CHANCERY LANE E`, `MCCRANEY ST E`, `MUNN'S AVE`. The
engine's `expand_street_name()` currently turns those into `CHANCERY LANE East`
and `MCCRANEY Street` — it was written for Toronto, whose source is already
title-cased, and it only rewrites the trailing suffix and direction tokens.
1,646 of 1,647 distinct Oakville street names are affected. **Every uploaded
`addr:street` would be wrong.**

- [ ] **11.** Write a title-casing step for source street names, applied before
      `expand_street_name()`. Do **not** use Python's `str.title()` — it breaks
      `STILL'S LANE` → `Still'S Lane` and `MCCRANEY` → `Mccraney`. The classes
      that occur in this dataset, with counts and examples, are tabulated in the
      source profile: apostrophe-S, `O'`, `Mc`, `Mac`, `ST.` for Saint, leading
      `THE`, and numeric names like `REGIONAL ROAD 25`.

- [ ] **12.** Add the type tokens the source spells out to the expansion table,
      or make the title-caser handle them: `LANE` (86 streets), `TRAIL` (49),
      `GATE` (40), `COMMON` (35), `WAY` (62), `LINE`. They currently pass
      through untouched and stay uppercase.

- [ ] **13.** **Settle the ambiguous forms against OSM, not against a rule.**
      `MacDonald` vs `Macdonald`, `St. Ann's` vs `Saint Ann's` — Oakville
      already has thousands of mapped streets, many touched by you. The engine's
      `/streets` page diffs normalized source names against OSM's `name` values;
      use it, and prefer what is already on the ground. Where the source is
      genuinely wrong, that is what the street-override table is for (a
      hardcoded per-city map — deliberately not a global `alt_name` lookup,
      which is unsafe across cities that reuse street names).

- [ ] **14.** Verify against all 1,647 distinct names before running anything
      wide. A test over the full list is cheap and this is the one error class
      that would be spread across tens of thousands of objects.

- [ ] **15.** **Verify the housenumber suffixes survive.** 24 addresses are
      `335A LAKESHORE RD E`-style. The config already routes around the
      tracker's suffix-dropping `number` column (`number_from = "full"`,
      Finding 2), but confirm those 24 rows come out as `335A` and not `335` —
      uploading `335` there would be wrong *and* collide with the real 335.

- [ ] **16.** **Verify the unit-embedded `full` column does no damage.**
      `ADDRESS` contains `... UNIT 33` on 5,896 rows (Finding 3). Collapse-to-civic
      should key on (number, street) and ignore it, but check the candidate
      count lands near 65,135, not near 71,049.

---

## Phase 3 — Prove it on the dev server

- [ ] **17.** Run one small tile end to end: Ingest → Fetch OSM → Conflate →
      Run checks → Review → Upload, all on DEV. The engine README's "First
      end-to-end run" walks the buttons.

- [ ] **18.** Read the whole review queue for that tile rather than
      click-approving. You are calibrating the checks — `match_far`,
      `city_duplicate`, `nearby_street_mismatch`, `missing_sample` — against a
      city you know personally, which is a luxury most of these imports don't
      have.

- [ ] **19.** Export a `.osm` and open it in JOSM. Look at the tags on real
      objects before any of them are real. Especially: `addr:street` casing,
      `addr:housenumber` on the suffixed rows, and the `source` tag.

- [ ] **20.** Decide the tag set for Oakville and write it down. Open
      questions: `addr:city=Oakville` on every node? `addr:province` — the
      family is inconsistent (Waterloo uses `ON`, Guelph `Ontario`); match what
      Oakville already has. No postcode is available — the source's `POSTAL`
      column is empty on every row.

- [ ] **21.** Settle the `source` tag. Your changesets currently carry
      `Esri World Imagery; Oakville Address Points (Open Government Licence -
      Town of Oakville)`. The engine writes `[export] attribution`, currently
      `Town of Oakville Open Data`. Make them agree, or diverge on purpose.

- [ ] **22.** Run several tiles. Confirm nothing degrades at volume and that
      resumability works — kill the process mid-run and restart it.

---

## Phase 4 — Community and governance (yours)

This is the part that is easy to skip and expensive to skip. Adding addresses
by hand through MapRoulette is ordinary mapping. Running a pipeline that
creates tens of thousands of nodes is an **import**, and the community treats
it differently — not because of the data, but because of the volume and the
irreversibility.

- [ ] **23.** **Read the OSM Import Guidelines** end to end if you haven't
      recently: <https://wiki.openstreetmap.org/wiki/Import/Guidelines>

- [ ] **24.** **Create a dedicated import account** (conventionally
      `<something>_Import`). Keeps your ordinary mapping history separate from
      bulk edits, and lets the community review or revert one without touching
      the other. This is a strong convention, not a rule, but people will ask.

- [ ] **25.** **Write the import plan** on the OSM wiki. Toronto's is the
      template: `toronto-2-address-import/IMPORT_PROPOSAL.mediawiki`. It should
      cover the source and its licence, the tag mapping, the conflation method
      (including its known failure modes), the goals and non-goals, and how you
      handle conflicts with existing data.

      Say plainly that you have been adding these addresses manually since
      2026-08 and that this is the same work continued at scale. That history
      is an asset — it makes you the local, not an outsider bulk-loading a city.

- [ ] **26.** **Announce on the `imports@openstreetmap.org` mailing list**, and
      give it real time for replies. Also worth posting to the Canadian
      community (OSM Canada Slack / community.osm.org) — `Matthew Darwin`,
      `zzptichka`, and `DannyMcD` are active on Ontario address imports and
      have opinions worth having early.

- [ ] **27.** **Deal with MapRoulette challenge 55881.** If the import
      supersedes it, say so on the challenge and wind it down; if other people
      are working it, they need to know an import is coming, or you will create
      duplicates against their in-flight work. Your call — you built it.

- [ ] **28.** **Fill `[export] import_plan`** in `config.toml` with the wiki URL.
      This unblocks the engine's upload path. Do it only after items 25 and 26
      are genuinely done.

---

## Phase 5 — Production

Do not start this phase until: **licence gate closed (skfd confirms)** and
**items 24–28 complete**.

- [ ] **29.** Register a second OAuth2 app on `openstreetmap.org` (production
      has its own registry) and fill `.env.prod` — separate `FLASK_SECRET_KEY`
      and `FERNET_KEY` from dev; they are not required to match.

- [ ] **30.** Launch with `python run.py --city-dir ../oakville-address-import --prod`
      and confirm the header says **PROD**.

- [ ] **31.** **Upload one small tile first.** Then stop. Look at it in iD, in
      JOSM, and in the changeset view. Let it sit for a day. This is the only
      moment where a mistake is still cheap.

- [ ] **32.** Scale up gradually — a ward at a time, not the town. The upload
      throttle is `[upload] changesets_per_minute = 1` for a reason.

- [ ] **33.** Watch for reverts, comments, and changeset discussion, and answer
      them quickly. One responsive reply prevents most import disputes.

- [ ] **34.** Publish a DB snapshot after the first production upload:
      `/publish-db` from the engine. `data/oakville/tool.db` is the only record
      of what has been pushed — if you lose it, you lose the ability to know
      what is already done.

---

## Ongoing

- [ ] **35.** Monthly maintenance: `python -m t2.maintenance` prints the delta
      since the watermark, `--prepare` ingests and conflates the additions. The
      source churns by single digits per pull, so this is small.

- [ ] **36.** A month is not closed until its DB snapshot is published — the
      engine enforces this.

---

## Things you may reasonably decide against

Written down so that changing your mind counts as a decision rather than a
drift:

- **Not importing at all.** If the gap turns out small and diffuse, the honest
  answer may be that manual entry plus the tool as a *QA layer* beats an
  import. Guelph landed exactly there (`guelph-address-import`) — brownfield,
  ~6% missing, no import proposal written on purpose.
- **Units.** The config collapses 71,049 rows to 65,135 civic addresses and
  defers the 5,896 unit rows. Importing units is a separate, later decision.
- **Skipping the dedicated account.** Defensible for a local mapper importing
  their own city, but expect to explain it.
