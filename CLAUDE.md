# Claude Code orientation — oakville-address-import

Read this before acting in this repo. It is written for an agent joining a
codebase whose important parts live somewhere else.

## What this repo is, and is not

This is a **thin city checkout**, not an application. It holds `config.toml`,
credential templates, onboarding evidence, and the import checklist. There is
no code here.

The code — the pipeline, the web UI, the conflation engine — lives in the
sibling repo **`address-importer-friend`** (the "engine"). Almost every change
you are asked to make will be *there*, not here. The expected layout:

```
Code/
  address-importer-friend/     # engine. Run commands from HERE.
  oakville-address-import/     # this repo, selected with --city-dir
  ontario-address-changes/     # the tracker: source data, per-dataset configs
```

Run the tool from the engine root, naming this checkout:

```bash
cd ../address-importer-friend
python run.py --city-dir ../oakville-address-import          # dev sandbox
python run.py --city-dir ../oakville-address-import --prod   # production
```

`T2_CITY_DIR` does the same job for the `python -m t2.*` entrypoints.

## Start with the checklist

`IMPORT_CHECKLIST.md` is the task list and its ordering is deliberate. When the
user asks for something vague ("let's work on Oakville"), find where they are on
that list rather than inventing a plan. When you finish an item, tick it and
say which one.

`onboarding/source-profile-2026-08-16.md` is the evidence base. Every
non-obvious line in `config.toml` traces to a numbered finding there. If you are
about to change `config.toml`, read that file first — several settings look
wrong until you know why they are that way.

## Traps specific to Oakville

These have already cost investigation once. Do not rediscover them.

1. **Street names are UPPERCASE.** 1,646 of 1,647 distinct names.
   `t2.conflate.expand_street_name()` only rewrites trailing suffix/direction
   tokens, so it emits `MCCRANEY Street`. A title-caser is needed and does not
   exist. **Never use `str.title()`** — it breaks `STILL'S LANE` and `MCCRANEY`.
   Finding 1 in the source profile has the full edge-case table.

2. **`number_from = "full"`, not `"number"`.** The tracker's `number` column
   drops housenumber suffixes: `335A LAKESHORE RD E` is stored as `335`. Only
   `ADDRESS` carries the combined form. This is why `full_from` is `"full"` and
   not the cleaner `"number+street"` — the engine rejects that pair as circular
   (`t2/config.py:147`). Changing one of these forces the other.

3. **`LABEL` is not the housenumber.** It equals `STREET_NUM` on plain rows but
   holds the *unit* on all 5,896 unit rows. It looks like a shortcut and is a
   trap.

4. **`ADDRESS` embeds the unit** (`1135 MCCRANEY ST E UNIT 33`). Any
   `DISTINCT full` count is inflated. Candidate counts should land near 65,135,
   not 71,049.

5. **`[export] import_plan` is empty on purpose** and the engine refuses to
   upload while it is. That is the gate holding production until the licence
   review and the import proposal are done. Do not fill it, suggest filling it,
   or work around it to make an error go away. If a user hits that error, the
   correct response is to explain the gate.

6. **The dev/prod distinction is real.** Default is the dev sandbox. Never add
   `--prod` to a command on your own initiative.

## Engine conventions worth knowing

- **Terminology.** A *candidate* (≡ *AddressMatch*) is one source row paired
  with its OSM lookup. It carries three orthogonal axes: `verdict` (what
  conflation decided — `MATCH`, `MATCH_FAR`, `MISSING`, `SKIPPED`), `status`
  (what the operator decided — `OPEN`, `APPROVED`, `REJECTED`, `DEFERRED`), and
  `stage` (where it sits in the pipeline). A *run* is one execution over one
  area and is also the unit of upload — one run becomes one changeset.
- **The audit log is append-only** and every automatic action writes to it.
  Preserve that property; do not add a code path that mutates state without an
  `audit.log()` call.
- **`data/` is gitignored and precious.** `data/oakville/tool.db` is the single
  source of truth for what this city has pushed to OSM. Never delete or
  reinitialize it to fix a problem. There is no re-ingest path — recovering
  from a bad ingest means a surgical reset plus re-conflation of the affected
  rows, not a wipe.
- **The tracker is read-only from here.** Do not write to
  `ontario-address-changes/data/oakville/oakville.db`. Its per-dataset config
  (`datasets/oakville.toml`) documents source-field decisions and is worth
  reading; changing it is a tracker-side decision.
- **Config changes are cheap; ingest re-runs are not.** Getting
  `[source_fields]` right before a wide run matters more than usual.

## House rules for changes

- Engine changes must stay **city-agnostic**. Oakville is the first city in the
  family whose source is uppercase, so a title-caser belongs in the engine
  behind config, not as an Oakville special case — but it must not change
  behaviour for cities whose sources are already title-cased. There are tests;
  run them.
- When a fix is genuinely Oakville-specific (a wrong street name in the source,
  say), the street-override table is the intended mechanism. It is deliberately
  a hardcoded per-city map — a global OSM `alt_name` lookup was considered and
  rejected as unsafe, because street names collide across cities.
- Prefer measuring to assuming. The tracker DB is a local SQLite file; a
  `sqlite3` query answers most "does the source really…" questions in seconds,
  and this repo's onboarding notes exist because someone ran those queries.
