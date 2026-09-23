# NOTES

Commands used to build/extract the basemap `.pmtiles`, generate `style.json`,
and deploy both to Hetzner Object Storage for tuulikartta.info's basemap.

Bucket/endpoint used throughout: `tuulikartta-tiles` @ `https://hel1.your-objectstorage.com`
(rclone remote `hetzner`, aws-cli profile `hetzner`, both already configured on this machine).

**Currently deployed:** `nordic-baltic.pmtiles` (mainland Iceland, Norway,
Sweden, Finland, Denmark, Estonia, Latvia, Lithuania — no Greenland, Faroe
Islands, or Svalbard — z0-12), built via extraction (§0). The original
`finland.pmtiles` (Finland only, z0-15, built from scratch with Planetiler,
§1) is superseded but left in place below as a documented alternative if
per-country full-detail builds are ever needed again.

## 0. Extract a region from Protomaps' public planet build (current approach)

Rather than reprocessing raw OpenStreetMap data yourself, `pmtiles extract`
can pull just a bounding box out of Protomaps' daily public planet-wide
build over HTTP range requests — no local Planetiler run, no 30+GB OSM
source download, no hours-long build. Trade-off: you get Protomaps' schema/
cadence as-is (which is what this project's style already targets anyway)
and whatever max zoom you choose, rather than fully custom processing.

Install the CLI:

```bash
brew install pmtiles
```

Extract (bbox is `min_lon,min_lat,max_lon,max_lat`; `--maxzoom` controls
both file size and how much has to be downloaded — each extra zoom level
roughly quadruples the number of tiles in a fixed region):

```bash
cd ~/Git/tuulikartta/basemaps/tiles/data
pmtiles extract https://latest.protomaps.com/v4.pmtiles nordic-baltic.pmtiles \
  --bbox=-25,53.5,29,71.5 --maxzoom=12
```

That bbox covers mainland Iceland through the North Cape and south to
Lithuania. Result: 1.5GB, ~5.5 minutes, 107 HTTP requests, 171k tiles — vs.
730MB source / ~5GB scratch disk / hours for the Finland-only Planetiler
build in §1 below, for 8 countries instead of 1 (at z12 instead of z15 —
no building-level detail, but full place/road labels and city-level detail).

Sanity-check before committing to a bigger extract: run a quick low-zoom
test first (`--maxzoom=6` finishes in seconds) to confirm the bbox/URL are
right, then scale up.

```bash
ls -lh nordic-baltic.pmtiles
xxd -l 16 nordic-baltic.pmtiles   # should start with "PMTiles", not zeros
```

Then generate/update `style.json`'s source url to point at wherever it's
uploaded (§2-3 below) — everything past this point is identical regardless
of whether the `.pmtiles` came from extraction or a from-scratch build.

## 1. Alternative: build finland.pmtiles from scratch with Planetiler (tiles/)

Clean out any previous build state first — a partially-finished build leaves
a corrupt `.pmtiles` (starts with zero bytes instead of the `PMTiles` magic
header) and a large `data/tmp/` scratch directory:

```bash
cd ~/Git/tuulikartta/basemaps/tiles
rm -f data/finland.pmtiles
rm -rf data/tmp
```

Build the image:

```bash
docker build -t protomaps/basemaps .
```

Run the build. Notes on the flags:
- `-v ./data:/tiles/data` — the leading `./` matters. `-v data:/tiles/data`
  (no path separator) makes Docker create/reuse a **named volume** called
  `data` instead of bind-mounting the host directory — the build then
  "succeeds" but writes into `/var/lib/docker/volumes/data/_data`, invisible
  on the host. Check with `docker inspect <container> --format '{{json .Mounts}}'`
  if output ever seems to go missing.
- `-e JAVA_TOOL_OPTIONS="-Xmx4g"` — without an explicit heap size the JVM's
  container-memory auto-detection undersizes the heap and the build dies
  with `java.lang.OutOfMemoryError: Java heap space` partway through. `4g`
  fits inside Docker Desktop's default ~7.7GB VM allocation with headroom
  left for the container OS and Planetiler's off-heap mmap'd files.
- Omitting `--download`/`--force` reuses the already-downloaded
  `data/sources/finland.osm.pbf` (730MB) instead of re-fetching it.

```bash
docker run -v ./data:/tiles/data -e JAVA_TOOL_OPTIONS="-Xmx4g" --rm -it protomaps/basemaps \
  --output=data/finland.pmtiles --area=finland
```

Or detached, so it survives a terminal disconnect (takes ~2-3 hours):

```bash
docker run -v ./data:/tiles/data -e JAVA_TOOL_OPTIONS="-Xmx4g" -d --name basemap-build protomaps/basemaps \
  --output=data/finland.pmtiles --area=finland
docker logs -f basemap-build
```

Verify the result is real (should be tens-to-hundreds of MB+ and start with
the `PMTiles` magic bytes, not zeros):

```bash
ls -lh data/finland.pmtiles
xxd -l 16 data/finland.pmtiles
```

### Recovery: output landed in a named volume instead of data/

If the `./` was dropped from the `-v` flag (see above), pull the file back
out of the stray named volume instead of rebuilding:

```bash
docker run --rm -v data:/check alpine ls -lh /check/
docker run --rm \
  -v data:/from \
  -v ~/Git/tuulikartta/basemaps/tiles/data:/to \
  alpine cp /from/finland.pmtiles /to/finland.pmtiles
docker volume rm data   # cleanup once copied out
```

### Docker disk hygiene

Worth checking before a build if disk is tight — old images/build cache can
account for tens of GB of reclaimable space:

```bash
docker system df
docker system prune -a --volumes
```

## 2. Generate style.json (styles/)

```bash
cd ~/Git/tuulikartta/basemaps/styles
npm ci
npm run generate_style style.json "pmtiles://https://tuulikartta-tiles.hel1.your-objectstorage.com/nordic-baltic.pmtiles" light fi
```

Positional args: output path, tile source URL (a `pmtiles://` URL works
directly, no separate TileJSON needed), flavor (`light`/`dark`/a named
flavor/a custom flavor file), language code (`fi` for Finnish labels).

The generated style's `glyphs`/`sprite` point at Protomaps' free public CDN
(`protomaps.github.io/basemaps-assets`) by default — no need to self-host
fonts or sprites alongside the tiles.

`basemaps/style.json` (repo root, not `styles/style.json`) is a
hand-adjusted variant using the `white` flavor — kept as the one actually
deployed. If regenerating from scratch, remember to point its `sources.protomaps.url`
at the real bucket, not a placeholder/demo URL.

## 3. Upload to Hetzner

```bash
cd ~/Git/tuulikartta/basemaps

rclone copyto tiles/data/nordic-baltic.pmtiles hetzner:tuulikartta-tiles/nordic-baltic.pmtiles \
  --s3-acl public-read \
  --header-upload "Content-Type: application/octet-stream" \
  --header-upload "Cache-Control: public, max-age=3600" \
  --progress --stats-one-line --stats 5s

rclone copyto style.json hetzner:tuulikartta-tiles/style.json \
  --s3-acl public-read \
  --header-upload "Content-Type: application/json" \
  --header-upload "Cache-Control: public, max-age=3600" \
  --progress
```

(Substitute whatever `.pmtiles` filename you actually built/extracted —
`finland.pmtiles`, `nordic-baltic.pmtiles`, etc.)

`--s3-acl public-read` is required for both — the bucket policy only grants
public read under the `basemap/*` prefix (left over from an earlier,
unrelated raw-tile-pyramid attempt), so root-level objects like these two
need their own public-read ACL to be fetchable.

## 4. Verify the deployed files

```bash
# style.json is reachable, right content-type, CORS present
curl -sD - -o /dev/null -H "Origin: https://tuulikartta.info" \
  https://tuulikartta-tiles.hel1.your-objectstorage.com/style.json

# the .pmtiles file supports range requests (required for PMTiles reads)
curl -sD - -o /dev/null -H "Range: bytes=0-15" \
  https://tuulikartta-tiles.hel1.your-objectstorage.com/nordic-baltic.pmtiles

# current bucket CORS / public-read policy
aws s3api get-bucket-cors --profile hetzner --endpoint-url https://hel1.your-objectstorage.com --bucket tuulikartta-tiles
aws s3api get-bucket-policy --profile hetzner --endpoint-url https://hel1.your-objectstorage.com --bucket tuulikartta-tiles
```

## Sizing reference

**Extraction (§0):** cost scales with region area × zoom, not with running
a build — no meaningful RAM/scratch-disk requirement, just network + the
output file size. Datapoints from this project: Nordic+Baltic bbox above
at maxzoom=6 → 10MB/6s; at maxzoom=12 → 1.5GB/5.5min. The whole planet at
z0-15 is documented at ~120GB, "each additional zoom level roughly doubles
the size" — so budget accordingly before jumping straight to z15 on a large
region; test at a low maxzoom first.

**From-scratch Planetiler build (§1):** rule of thumb (confirmed against
the Finland build): **disk ≈ 1GB + 5-10× the `.osm.pbf` size**, **RAM ≈
0.5× the `.osm.pbf` size** minimum (in practice give the JVM heap more —
see `-Xmx4g` above). Finland's extract is 730MB (→ ~4.5-8GB scratch disk,
1.7GB finished file at z15). For scale, Europe's Geofabrik extract alone is
~33GB, implying ~165-330GB of scratch disk — impractical on a laptop with
limited free space, which is exactly why §0's extraction approach is used
for anything larger than one country.
