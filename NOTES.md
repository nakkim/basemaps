# NOTES

Commands used to build `finland.pmtiles`, generate `style.json`, and deploy
both to Hetzner Object Storage for tuulikartta.info's basemap.

Bucket/endpoint used throughout: `tuulikartta-tiles` @ `https://hel1.your-objectstorage.com`
(rclone remote `hetzner`, aws-cli profile `hetzner`, both already configured on this machine).

## 1. Build finland.pmtiles (tiles/)

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
npm run generate_style style.json "pmtiles://https://tuulikartta-tiles.hel1.your-objectstorage.com/finland.pmtiles" light fi
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

rclone copyto tiles/data/finland.pmtiles hetzner:tuulikartta-tiles/finland.pmtiles \
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

`--s3-acl public-read` is required for both — the bucket policy only grants
public read under the `basemap/*` prefix (left over from an earlier,
unrelated raw-tile-pyramid attempt), so root-level objects like these two
need their own public-read ACL to be fetchable.

## 4. Verify the deployed files

```bash
# style.json is reachable, right content-type, CORS present
curl -sD - -o /dev/null -H "Origin: https://tuulikartta.info" \
  https://tuulikartta-tiles.hel1.your-objectstorage.com/style.json

# finland.pmtiles supports range requests (required for PMTiles reads)
curl -sD - -o /dev/null -H "Range: bytes=0-15" \
  https://tuulikartta-tiles.hel1.your-objectstorage.com/finland.pmtiles

# current bucket CORS / public-read policy
aws s3api get-bucket-cors --profile hetzner --endpoint-url https://hel1.your-objectstorage.com --bucket tuulikartta-tiles
aws s3api get-bucket-policy --profile hetzner --endpoint-url https://hel1.your-objectstorage.com --bucket tuulikartta-tiles
```

## Sizing reference

Planetiler's own rule of thumb (confirmed against this Finland build):
**disk ≈ 1GB + 5-10× the `.osm.pbf` size**, **RAM ≈ 0.5× the `.osm.pbf` size**
minimum (in practice give the JVM heap more — see `-Xmx4g` above). Finland's
extract is 730MB, giving a scratch-disk need of roughly 4.5-8GB; the finished
`finland.pmtiles` came out to 1.7GB.
