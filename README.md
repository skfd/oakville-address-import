# oakville-address-import

The **Oakville city checkout** of the address-import family, scaffolded
2026-08-16 for [`TronnaLegacy`](https://www.openstreetmap.org/user/TronnaLegacy),
who has been adding Oakville addresses from the town's open data by hand since
2026-08-08 (MapRoulette challenge 55881).

**Start here: [`IMPORT_CHECKLIST.md`](IMPORT_CHECKLIST.md).** It is the ordered
to-do list, and it names which gates are yours and which are not.

Status: **onboarding, brownfield-active.** No conflation run has been executed
yet. Nothing has been uploaded by this tool, to any server.

## What this is

The pipeline lives in the engine repo,
[`address-importer-friend`](https://github.com/skfd/address-importer-friend):
it reads a city's address points from a tracker database, conflates them
against live OSM data, routes anything questionable to a human reviewer through
a local web UI, and uploads what you approve — with an append-only audit log of
every decision, automatic or manual.

This repo carries only Oakville's part: `config.toml`, credential templates
(`.env.*`, gitignored), the onboarding probe evidence, and — locally, never
committed — `data/` with the OSM extract and `data/oakville/tool.db`.

```bash
# from the engine repo, with this checkout as a sibling directory
cd ../address-importer-friend
python run.py --city-dir ../oakville-address-import
```

Toronto is the family's completed import (~449k addresses, 2026-05); Hamilton,
Guelph and a dozen other Ontario cities are onboarded at various depths. Each
one taught the engine something, and Oakville inherits all of it.

## The source

Town of Oakville **Address Points**, layer 15 of
`maps.oakville.ca/oakgis/rest/services/ArcGISOnline/AGO_Oakville_Misc_Layers`,
consumed through the sibling
[`ontario-address-changes`](https://github.com/skfd/ontario-address-changes)
tracker (`data/oakville/oakville.db`).

**71,049 active rows** at snapshot 26 (2026-08-12) → **65,135 civic addresses**
after collapsing the 5,896 unit rows. Tiles are built from the town's **7 ward
polygons**, which point-test at **100.00%** coverage — the best in the
portfolio.

Full probe results, and the reasoning behind every setting in `config.toml`:
[`onboarding/source-profile-2026-08-16.md`](onboarding/source-profile-2026-08-16.md).

## Two things to know before you touch anything

**1. Street names are UPPERCASE and the engine cannot fix them yet.**
The source publishes `CHANCERY LANE E` and `MCCRANEY ST E`; the current
normalizer turns those into `CHANCERY LANE East` and `MCCRANEY Street`. 1,646
of 1,647 distinct street names are affected. This is a code fix, it is Phase 2
of the checklist (items 11–16), and it blocks any upload — to dev as well, if
you want the dev runs to mean anything.

**2. Production upload is gated on a licence review you don't have to chase.**
Oakville's Open Government Licence is a clean clone of OGL-Canada 2.0 and has
been verified as ODbL-compatible on a desk review, but the confirmation from
the OSM Licensing Working Group is still outstanding. **skfd owns that.** The
dev sandbox is unaffected — run there as much as you like.

The engine enforces this with a real mechanism rather than a comment:
`[export] import_plan` in `config.toml` is empty, and changeset-tag
construction raises while it is — which blocks the tool's own upload and the
JOSM `.osm` export alike, so there is no accidental side door.

## Status of the surrounding work

- The town's licence sits in a drafted (unsent) letter to the OSM Licensing
  Working Group covering six Ontario OGL clones. skfd is handling it.
- A **hosted instance** of the review UI — so you can run all of this from a
  browser instead of a local Python install, signing in with your own OSM
  account — is designed but not built:
  [`future-work/hosted-instance.md`](https://github.com/skfd/address-importer-friend/blob/main/future-work/hosted-instance.md)
  in the engine repo. Until then it runs locally.

## Working with Claude Code

This repo has a [`CLAUDE.md`](CLAUDE.md) that orients Claude Code on the layout,
the engine's conventions, and the traps specific to Oakville. Point it at the
checklist and it can do most of Phase 2 with you.

MIT licensed. The address data is the Town of Oakville's, under its own
Open Government Licence.
