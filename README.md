> [!IMPORTANT]
> **This repository has been consolidated into the unified [`earthwatch-suite`](https://github.com/crieck2010/earthwatch-suite) monorepo.**
> Development, CI, and new features continue at [crieck2010/earthwatch-suite](https://github.com/crieck2010/earthwatch-suite).

# Imagery Atlas

A satellite imagery source atlas for the [Earthwatch](https://github.com/crieck2010/earthwatch-suite)
remote-sensing stack. It answers one question: **for any job, which imagery source should the
pipeline pull, and how?**

Open `index.html` in a browser — it is fully self-contained and works offline. The same file
doubles as the desktop version: download it and double-click.

## What's inside

- **Ranked free-source table** — every ingestible free source ordered by resolution × revisit,
  with the best use case per source.
- **18 source dossiers** — GSD per band group, spectral bands with center wavelengths,
  revisit and coverage, programmatic access (STAC catalog, collection ID, signup / keys /
  rate limits), commercial-use license terms, and ingestion quirks.
- **Commercial high-resolution tier** — Planet, Maxar, Airbus, and commercial SAR, with
  pricing models where publicly stated.
- **Best-source-per-job selection logic** — vegetation, water, burn, coast, thermal,
  elevation change: which source wins and why.
- **Python/STAC ingestion playbook** (`docs/INGESTION.md`) — endpoints, collection IDs,
  and the radiometric scaling rules the engines actually use.

## Data snapshot

Research snapshot: **2026-09-25**, verified live against Element84 Earth Search and
Microsoft Planetary Computer `raster:bands` metadata. Satellite programs evolve —
re-verify collection IDs and scaling against the catalogs before trusting a new scene.

Key scaling rules baked into the Earthwatch engines (see also `docs/INGESTION.md`):

| Product | Scale | Offset |
|---|---|---|
| Sentinel-2 L2A Collection 1 (surface reflectance) | 0.0001 | −0.1 |
| Landsat C2 L2 surface reflectance | 0.0000275 | −0.2 |
| Landsat C2 L2 surface temperature (ST_B10) | 0.00341802 | +149 K |

Always prefer per-asset `raster:bands` scale/offset from the STAC item over hardcoded
values — the engines in `survey-imagery` and `survey-thermal` do exactly that, with the
table above as fallback.

## Web app

The interactive version (live search, filters, sortable comparison, collection-ID lookup,
and a "pick my source" chooser) is hosted in the Muse app; this repo carries the full
reference page and the ingestion playbook.

## Versions

See [CHANGELOG.md](CHANGELOG.md). `v1.0.0` ships the 2026-09-25 research snapshot.

## License

MIT — see [LICENSE](LICENSE).
