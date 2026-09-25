# Python/STAC Ingestion Playbook

How the Earthwatch stack pulls satellite imagery programmatically. Snapshot verified
live **2026-09-25**; re-verify collection IDs against the catalogs before trusting a
new scene.

## STAC endpoints

| Catalog | Endpoint | Access notes |
|---|---|---|
| Element84 Earth Search | `https://earth-search.aws.element84.com/v1` | Anonymous discovery + reads for most COG collections |
| Microsoft Planetary Computer | `https://planetarycomputer.microsoft.com/api/stac/v1` | Open discovery; some assets need signed URLs / free account |
| Copernicus Data Space | `https://stac.dataspace.copernicus.eu/v1` | Authoritative Copernicus catalog; free account commonly needed for asset reads |
| USGS LandsatLook | `https://landsatlook.usgs.gov/stac-server` | Note: `/stac-server`, not `/stac` |
| NASA CMR STAC | `https://cmr.earthdata.nasa.gov/stac` | MODIS, VIIRS, ASTER, HLS, hyperspectral, GEDI, ICESat-2; Earthdata credentials may be needed |

## Collection IDs and scaling

| Collection (catalog) | Contents | Scale | Offset | Notes |
|---|---|---|---|---|
| `sentinel-2-c1-l2a` (Earth Search) | Sentinel-2 L2A Collection 1 COGs | 0.0001 | **−0.1** | Current default. Assets use lowercase names (`red`, `green`, `nir`, …) |
| `sentinel-2-pre-c1-l2a` (Earth Search) | Sentinel-2 pre-C1 L2A COGs | 0.0001 | −0.1 | Same scaling |
| `sentinel-2-l2a` (Earth Search, Sinergise legacy) | Sentinel-2 L2A from 2015-06-27 | 0.0001 | **0.0** | Legacy offset — know which collection you queried |
| `landsat-c2-l2` (Earth Search / MPC) | Landsat 4–9 C2 L2 | 0.0000275 | −0.2 | Surface reflectance assets |
| `landsat-c2-l2` ST assets (`lwir`, `lwir11`) | Landsat C2 surface temperature | 0.00341802 | **+149 K** | Kelvin = DN × 0.00341802 + 149. Nodata 0 |

**Rule: read per-asset `raster:bands` scale/offset from the STAC item and prefer it
over the table.** Catalog asset keys differ per catalog (Earth Search: `red`;
Planetary Computer: `B04`; USGS: `SR_B4`) — normalize with a per-catalog adapter,
which is what `survey-imagery`'s band registry does.

## Minimal pull pattern (pystac-client)

```python
from pystac_client import Client

client = Client.open("https://earth-search.aws.element84.com/v1")
search = client.search(
    collections=["sentinel-2-c1-l2a"],
    bbox=[-76.2, 43.0, -76.0, 43.2],   # AOI
    datetime="2026-09-01/2026-09-25",
    query={"eo:cloud_cover": {"lt": 20}},
)
items = list(search.items())
item = items[0]
red = item.assets["red"]
# Per-asset scaling from the item itself — prefer this over the table:
band = red.extra_fields["raster:bands"][0]
scale, offset = band["scale"], band["offset"]
href = red.href  # COG — stream the window you need, don't download the tile
```

## Source-selection logic (best data for the job)

- **Vegetation:** Sentinel-2 (10–20 m red edge) for detail; HLS for dense harmonized
  time series; MODIS/VIIRS for long regional baselines; hyperspectral for
  species/chemistry questions.
- **Water / flood:** Sentinel-2 for clear-sky extent; Sentinel-1 RTC under cloud;
  Landsat DSWE for historical water; Sentinel-3 for broad water quality.
- **Burn / fire:** GOES/VIIRS/MODIS for rapid detection; Sentinel-2/Landsat dNBR for
  severity; Sentinel-1 under persistent cloud.
- **Coast:** Sentinel-2 for fine optical shorelines; Sentinel-1 for cloud-resistant
  waterline support; tide metadata/correction is essential.
- **Thermal:** Landsat ST products for detail; Sentinel-3/MODIS/VIIRS for frequent
  regional monitoring; GOES for rapid heat/fire development.
- **Elevation change:** Copernicus DEM as baseline; GEDI/ICESat-2 as sparse truth;
  Sentinel-1 SLC/InSAR or repeat lidar for actual change. Never difference DEMs from
  different sources without coregistration and datum checks.

## Resolution priority (with judgment)

Highest resolution first, *except* when another source wins the job: HLS beats
Sentinel-2 on temporal density; Sentinel-1 beats everything optical under cloud;
Landsat wins thermal detail and history; GOES/VIIRS win rapid event detection.
NAIP (0.6 m aerial, CONUS, public domain) is the free very-high-resolution baseline
and training-label source in the US. Paid sub-meter (Maxar, Airbus, SkySat, Planet)
enters the picture when finer detail changes the product's value.

## Commercial-use licenses (snapshot)

- Copernicus (Sentinel-1/2/3): open terms allow commercial use.
- Landsat, MODIS, VIIRS, NAIP, HLS: public domain / open.
- ASTER: open archive; SWIR failed 2009; thermal operations ended 2026-01-16.
- PRISMA / EnMAP: check mission-specific terms before commercial use.
- Planet NICFI mosaics: non-commercial — do not build a commercial dependency.
- Commercial providers (Planet, Maxar, Airbus, ICEYE, Capella, Umbra): paid,
  API-accessible, generally restrict redistribution.
