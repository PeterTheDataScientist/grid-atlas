# Grid Atlas

**Where can you site a factory, a cold store, a data centre or a base station and know what the power will actually do? This measures night-time electricity availability for African cities, each against its own history, from twelve years of satellite imagery.**

**[Open the dashboard](https://peterthedatascientist.github.io/gridtruth/)**

[![License: Apache 2.0](https://img.shields.io/badge/code-Apache%202.0-blue.svg)](LICENSE)
[![Data: CC BY 4.0](https://img.shields.io/badge/data-CC%20BY%204.0-green.svg)](data/LICENSE)

---

Anyone deciding where to put capital in Africa has to price electricity availability, and there is no comparable, open series to price it from. Utility reporting is not comparable across borders: different definitions of an interruption, different reporting periods, different publication habits, and in many markets nothing machine-readable at all. So a decision that turns on power availability gets made on anecdote, or on a diesel line item somebody guessed.

The VIIRS Day-Night Band has imaged every square kilometre of Africa every night since January 2012, at 500 m, with one instrument and one calibration. That is the only dataset in existence that is consistent across every African market, and it is free.

## What it measures, and what it refuses to measure

**Every city is compared to its own history. Never to another city.**

That distinction is the entire project. A map coloured by absolute brightness is a map of population density and wealth: it would look authoritative and mean nothing. What this asks instead is *is this place darker than it normally is*, which is answerable for Harare and Lagos and a small town alike, because each is only ever compared to itself.

The measurement unit is the city's **own lit footprint**, not a box drawn around its centre. Each pixel's normal level is the 90th percentile of that pixel across all observations; pixels that never light up are excluded. The headline number, `dark_share`, is the fraction of that footprint sitting below half its own normal on a given night.

Using a box instead of a footprint is not a small error. It made Cape Town read five times dimmer than Johannesburg, because a quarter of Cape Town's box is ocean. That is written up in [FAILURES.md](FAILURES.md).

## Who this is for

Site selection and expansion teams pricing standby power before committing to a location. Telecoms and tower companies specifying batteries and generators per site. Cold chain operators in food and pharmaceutical distribution quantifying spoilage exposure. Solar and storage developers sizing systems against measured conditions rather than a customer's recollection. Investors, insurers and development finance institutions who need an availability series measured the same way in every market they look at.

## Status: preview, not a finding

26 cities, sampled at every new moon from 2022 onward, collection ongoing.

Lunar brightening is controlled by sampling near new moon and recording the phase on every observation. Poor-coverage nights are discarded. **Thin cloud is not corrected for**, and thin cloud dims a city exactly the way a partial outage does. A single low night is more likely weather than a power event.

The pre-registered validation gate is in [EVALUATION.md](EVALUATION.md): if `dark_share` cannot separate labelled South African load shedding nights at AUC 0.70 on held-out cities, the electricity interpretation is withdrawn publicly and this ships as what it demonstrably is, an open record of how African cities' brightness varies against their own baselines.

## Known bias, stated up front

Generators and rooftop solar dim less during an outage, and they track wealth. This systematically under-detects outages in richer areas. Any published result carries that caveat or it is dishonest.

## Scope

This is a planning and siting dataset. It measures what a satellite observed and makes no claim about cause. Generation capacity, transmission condition, maintenance, weather and demand all move the same number, and night lights cannot separate them. It is not a scorecard for any utility or any country, and using it as one goes beyond what the measurement supports.

## Data

| Source | What | Licence | Account |
|---|---|---|---|
| World Bank Light Every Night, `s3://globalnightlight` | Nightly VIIRS DNB radiance, 2012 to Dec 2025, 15 arc-second COGs | Public domain | **none** |
| NASA VNP46A2 via LAADS | Daily, cloud and lunar corrected, ~2 day lag | Open | free Earthdata login |

The open archive needs no credentials at all. Granules are Cloud Optimized GeoTIFFs already on the standard grid, so a geographic window is read over HTTP without downloading the file: a 250 MB granule costs a few hundred kilobytes.

A secondary watcher records published supply notices where any exist, so that if a schedule is ever published it is captured. It has found none so far, which is why the satellite layer is the record rather than a check on someone else's record.

**Real time is not available from any source.** Two days is the floor. Availability is a track-record question, so that costs nothing.

## Run it

```bash
pip install rasterio numpy
python pipeline/extract.py pipeline/places.json 20250815,20251204 pixels.npz
python pipeline/aggregate.py pixels.npz pipeline/places.json data/places.json
```

No API key. No account.

## Repository map

```
pipeline/          extraction from the open archive, and the footprint aggregation
src/gridwatch/     the secondary watcher for any published supply notices
data/              derived per-city series, CC BY 4.0
docs/              the dashboard, served by GitHub Pages
.github/           scheduled collection, resumable and checkpointed
```

## Documents

- [DESIGN.md](DESIGN.md), the method, what can break it, decisions and what was rejected
- [EVALUATION.md](EVALUATION.md), the pre-registered gate and what gets measured
- [OBJECTIONS.md](OBJECTIONS.md), ten arguments against this work with honest answers
- [FAILURES.md](FAILURES.md), what was tried that did not work

## Licence

Code Apache 2.0. Derived data CC BY 4.0. Both chosen so a company can build on this without asking anyone.

## Commercial use

Free forever under the licences above. For hosted deployment, a private feed, or custom regional analysis, contact [petermundowa.com](https://petermundowa.com).

## Citation

```bibtex
@software{mundowa_grid_atlas,
  author = {Mundowa, Peter Tinashe},
  title  = {Grid Atlas: night-time electricity availability for African cities, measured from satellite},
  url    = {https://github.com/PeterTheDataScientist/grid-atlas},
  year   = {2026}
}
```
