# Blind Spot

> **Hackathon project.** Built by Kissa Zehra together with my teammate [Aiza Gazyani](https://github.com/Aiza166) at the NASA Space Apps Challenge 2026 (team Hehehehehe, Karachi). This is my copy with the full commit history; the original repository is [Aiza166/BlindSpot-FIRMS](https://github.com/Aiza166/BlindSpot-FIRMS).

**A harmonized MODIS + VIIRS burning calendar for Pakistan and northern India, which says when the correction can't be trusted.**

[**Live app: blindspot-firms.pages.dev**](https://blindspot-firms.pages.dev) · NASA Space Apps Challenge 2026 · Challenge: *Harmonization of MODIS and VIIRS Hot Spots* · Team **Hehehehehe**: Aiza Gazyani and Kissa Zehra

![Explorer: year-by-month burning calendar with the sensor eras shaded](docs/screenshots/explorer-desktop-light.png)

## The problem

Every autumn farmers across Punjab, in Pakistan and India, burn crop stubble, and district disaster and air-quality officers need to know whether this season is worse than usual. NASA's satellite fire record goes back to 2003, but in 2012 it moved from the MODIS sensor to the sharper VIIRS. Pakistani Punjab's October–November fire count jumped **+613%** at once (+371% in Indian Punjab). Read as it is, that looks like a burning crisis. It is a change of instrument.

## What Blind Spot does

- **Puts MODIS and VIIRS on one scale.** The 2012 jump shrinks to **+12.7%** (Pakistan) and **-6.1%** (India).
- **Burning calendar** for any of 347 districts, either Punjab, or a box or circle drawn on the map: one row per year, one cell per month, raw or harmonized, from 2003 to October 2026.
- **"Is this month unusual here?"** The latest month against that place's median for the same month, cross-checked with a second satellite (NOAA-20). The current month comes in live from NASA's near-real-time feed, labelled *provisional*.
- **Critical period:** the months when burning usually starts, peaks and ends.
- **Flags the years it can't vouch for**, with the reason in plain words.
- **CSV and PNG downloads**, in colour-blind-safe colours (cividis; flags also use icons, stripes and text).

## Key findings

Numbers are generated from the results files (`data/processed/results/*.json`) by the pipeline; full detail in [`docs/RESULTS_T1.md`](docs/RESULTS_T1.md), [`RESULTS_T2.md`](docs/RESULTS_T2.md) and [`RESULTS_T3.md`](docs/RESULTS_T3.md).

- **The 2012 jump is a sensor artefact, and the correction removes it** (window 15 Oct–30 Nov 2012): +613% → +12.7% in Pakistani Punjab, +371% → -6.1% in Indian Punjab.
- **After 2021 the correction fails in Indian Punjab** (T1 held-out: 1 of 4 test years inside the 95% interval). The app flags 2022–2025 there (2023–2025 in Pakistani Punjab).
- **Pakistani Punjab T2 (is the Aqua/S-NPP ratio drifting?) is inconclusive**; Indian Punjab shows no drift on matched days.
- **Why it fails (T3):** burning that moved later in the day, past the afternoon satellites, is supported for Indian Punjab, but we cannot separate later burning from less burning.
- **Example:** Sangrur, November 2021 (an unflagged year) is 1.74× its November median, and NOAA-20 agrees (1.94×). November 2025 looks far lower (0.05×), but it is in a flagged year, so it is not read as "far fewer fires".

The tests and thresholds were fixed before the data were run (`docs/DECISIONS.md`, D-028, D-039). One T2 rule was changed after seeing data; both verdicts are reported.

## How it works

1. **Ingest** NASA FIRMS active-fire detections (MODIS Terra/Aqua, VIIRS S-NPP and NOAA-20); keep vegetation fires, mask year-round heat sources such as brick kilns, count by local solar date.
2. **Calibrate** an Aqua-MODIS → S-NPP-VIIRS factor per region and month on 2012–2021 only; before 2012 the record is Aqua × factor, after it VIIRS as observed.
3. **Test** out of time on held-out 2022–2025: T1 (does the 2012 jump go away?), T2 (has the ratio drifted?), T3 (if it fails, why?).
4. **Publish** static app data and serve it with a Next.js app (MapLibre, DuckDB-WASM in the browser). A single Cloudflare Pages Function fetches the current month from FIRMS near-real-time; the FIRMS key stays on the server.

Methods and limits are also on the app's [Methods](https://blindspot-firms.pages.dev/methods/) and [Results](https://blindspot-firms.pages.dev/results/) pages.

## Repository layout

| Path | What |
|---|---|
| `pipeline/blindspot/` | Python 3.11 pipeline: download, ingest, mask, calibrate, tests T1–T3, app data, page text |
| `tests/` | pytest suite (fixtures only, no network) |
| `web/` | Next.js static export, MapLibre, DuckDB-WASM; `web/functions/` is the near-real-time Pages Function |
| `web/public/data/` | The app's data (tracked), built by `make appdata` |
| `data/` | Raw and processed data (raw is gitignored, rebuilt by `make data`; small results JSON is tracked) |
| `docs/` | Plan, decisions, results T1–T3, data schema, deploy, references, project page text, video script |
| `docs/screenshots/` | Screenshots from the browser tests |
| `docs/video/` | The submitted 30-second demo video, its subtitles and sources |
| `ai-log.md`, `decisions.md` | Where AI was used, and what was decided and why, during the hackathon |
| `CLAUDE.md` | Project rules: the scientific rules every change must follow |

## Run it locally

Needs Python 3.11, Node 20+, and network access to NASA FIRMS and GitHub (for boundaries).

```bash
make install      # venv + dependencies
make test         # lint + pytest
make web          # build and serve http://localhost:4173
make web-test     # lint, types, unit tests, Playwright browser checks
make data         # optional: re-run the science from raw NASA data (about an hour, ~1.3 GB)
```

All commands are in the `Makefile`; `CLAUDE.md` describes each one. On Windows there is no `make`: see [`docs/WINDOWS.md`](docs/WINDOWS.md). Deployment: [`docs/DEPLOY.md`](docs/DEPLOY.md). The live month needs a FIRMS MAP_KEY as a server secret; copy `.env.example` to `.env` for local use and never commit it.

## Data sources and licences

| Data | Source | Licence / terms |
|---|---|---|
| MODIS C6.1 (Terra, Aqua), VIIRS S-NPP and NOAA-20 375 m active fires | NASA FIRMS (LANCE, ESDIS) | NASA open data |
| Pakistan ADM1/ADM2 boundaries | geoBoundaries gbOpen | Public Domain |
| India ADM1 boundaries | geoBoundaries gbOpen (DataMeet) | CC BY 2.5 IN |
| India ADM2 boundaries | geoBoundaries gbOpen (Pathways Data) | **ODbL 1.0**; our simplified `districts.geojson` is shared under ODbL 1.0 too |

Details: `web/public/data/ATTRIBUTION.md` and the app's [Attribution](https://blindspot-firms.pages.dev/attribution/) page. DOIs and references: [`docs/REFERENCES.md`](docs/REFERENCES.md).

We acknowledge the use of data and imagery from LANCE FIRMS operated by NASA's Earth Science Data and Information System (ESDIS). Boundaries: geoBoundaries (Runfola et al. 2020).

## Limitations

- A hotspot count is an index of burning seen at satellite overpass times, not burned area or emissions.
- Indian Punjab fails the held-out check after 2021; Pakistani Punjab after 2021 is untested (T2 inconclusive).
- Later burning and less burning cannot be separated with this data.
- No independent check against non-FIRMS data.
- Tested only in Punjab; another region needs its own calibration and the same three tests.
- 2026 is near-real-time from August, provisional, and not yet covered by the tests.

## Project history

The data pipeline, its tests and a first version of the app were prepared locally before the event; the team lead was told by the Universal Event Local Lead that local work, data download and a written plan were allowed. This repository was created when the hackathon opened on 14 Nov 2026 and starts from one commit that imports that work.

**Built during the hackathon (14–15 Nov 2026):** the 2012 story view; January to October 2026 added (`pipeline/blindspot/extend.py`, leaving 2003–2025 unchanged); the deploy to Cloudflare Pages with the live current month; a faster, retrying near-real-time function; wording fixes with tests; `ai-log.md` and `decisions.md` throughout. The version submitted for judging is tagged [`submission-2026-11-15`](https://github.com/Aiza166/BlindSpot-FIRMS/tree/submission-2026-11-15).

## AI use

This project used Super Intelligence (SI) tools, as the NASA Space Apps 2026 rules require us to state. Claude Code (Anthropic), including subagents, wrote most of the code and tests, ran code, security and accessibility reviews, and drafted the documents and the video script. The demo video was assembled with AI from real screen captures of the live app, with AI-generated narration; every frame carries a visible label. People chose the question, scope and tests and approved each change; every number is read from pipeline output by a script. Full statement: "Use of Super Intelligence (SI)" in [`docs/SUBMISSION.md`](docs/SUBMISSION.md); also noted in `pyproject.toml` and `web/package.json`.

## Licence

No licence chosen yet (no `LICENSE` file), so all rights are reserved by default. Data files carry their own licences (above).
