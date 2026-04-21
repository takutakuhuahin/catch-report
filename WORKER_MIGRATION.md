# Worker migration — accept client-side GPS fields, drop EXIF dependency

The catch-report client (GitHub Pages) now sends EXIF-stripped images and passes GPS coordinates as separate FormData fields. The Worker (`line-harness.gyotak.workers.dev`) must be updated to accept the new fields and persist them to D1.

Until the Worker is updated, the new fields are silently ignored — the upload still succeeds with the EXIF-stripped image (so GPS leakage via R2 is stopped immediately), but GPS is not stored anywhere.

## D1 schema change

Add three nullable columns to `catch_reports`:

```sql
-- 0001_add_gps_columns.sql
ALTER TABLE catch_reports ADD COLUMN gps_lat REAL;
ALTER TABLE catch_reports ADD COLUMN gps_lng REAL;
ALTER TABLE catch_reports ADD COLUMN gps_alt REAL;
```

`REAL` (SQLite/D1 floating-point) rather than `TEXT` so future queries can do bounding-box filters. Columns are nullable so legacy rows without GPS remain valid.

Apply with:

```bash
wrangler d1 execute <DB_NAME> --file=./migrations/0001_add_gps_columns.sql --remote
```

If the existing schema has only one row per report but multiple photos per report (1:N images), put the GPS on the report row — use the first photo's GPS (or any non-null) per report. If each photo has its own row (e.g. `catch_report_images`), add the columns there instead.

## POST handler — field contract

The client's `generate()` call now sends FormData:

| field | cardinality | type | notes |
|---|---|---|---|
| `image` | N (≥1) | File (JPEG, EXIF-stripped) | existing |
| `comment` | 1 | string | existing |
| `gps_lat` | N (matches `image`) | string (decimal degrees) or `""` | new; empty if missing |
| `gps_lng` | N | string (decimal degrees) or `""` | new |
| `gps_alt` | N | string (meters) or `""` | new; null if camera omitted altitude |
| `photo_taken_at` | N | string (`"YYYY:MM:DD HH:MM:SS"`) or `""` | new; EXIF DateTimeOriginal or file mtime fallback |

The per-photo fields are appended once per `image`, in the same order, so `formData.getAll('gps_lat')[i]` is the GPS for `formData.getAll('image')[i]`.

## Worker handler outline

```js
// inside the POST /gyotak/catch-report handler
const form = await request.formData();
const images = form.getAll('image');
const lats  = form.getAll('gps_lat');
const lngs  = form.getAll('gps_lng');
const alts  = form.getAll('gps_alt');
const takens = form.getAll('photo_taken_at');

const parseNum = (v) => {
  if (v == null) return null;
  const s = String(v).trim();
  if (s === '') return null;
  const n = Number(s);
  return Number.isFinite(n) ? n : null;
};

// pick the first non-null as the report-level GPS
let gpsLat = null, gpsLng = null, gpsAlt = null;
for (let i = 0; i < images.length; i++) {
  const lat = parseNum(lats[i]);
  const lng = parseNum(lngs[i]);
  if (lat != null && lng != null) {
    gpsLat = lat; gpsLng = lng; gpsAlt = parseNum(alts[i]);
    break;
  }
}

// ... existing R2 put, hashing, reverse-geocode, article generation ...

await env.DB.prepare(
  'UPDATE catch_reports SET gps_lat = ?, gps_lng = ?, gps_alt = ? WHERE id = ?'
).bind(gpsLat, gpsLng, gpsAlt, catchReportId).run();
```

## Backward compatibility

- Old clients that do not send `gps_lat/gps_lng/gps_alt/photo_taken_at` will have `form.getAll('gps_lat')` return `[]`. The `parseNum` + `images.length` loop skips those iterations, so `gpsLat/gpsLng/gpsAlt` stay `null` — the D1 write stores NULL and the row remains valid.
- Old clients sent the raw EXIF-bearing image. The Worker's existing EXIF-based GPS path (if any) can stay as a fallback: use it only when `gpsLat` is null after the new path. Do NOT read EXIF from a new-client image — it is guaranteed stripped.
- Recommendation: after the Worker is deployed and all clients have updated (cache-busted GitHub Pages), remove the legacy EXIF-GPS code path. Until then, keep it gated behind `if (gpsLat == null) { /* legacy EXIF fallback */ }`.

## Reverse-geocoding

The existing Worker reverse-geocodes GPS → human-readable location (`img.location`, `img.locationEn`). Feed the client-supplied `gpsLat/gpsLng` into that same function instead of EXIF-parsed GPS.

## `photo_taken_at`

Persist alongside the photo/report. Use for display (frontend `img._exifDateTime` already consumes it) and for chronological sorting. If stored on the report row, a single column `photo_taken_at TEXT` suffices.

```sql
ALTER TABLE catch_reports ADD COLUMN photo_taken_at TEXT;
```

## Smoke test after Worker deploy

1. From the updated client: take a photo with GPS.
2. Confirm UI shows `🟢 GPS OK / 🔒 GPS座標を記録しました(非公開) / 写真からは EXIF 剥離済`.
3. Submit. Then:
   - `wrangler d1 execute <DB> --command="SELECT id, gps_lat, gps_lng, gps_alt, photo_taken_at FROM catch_reports ORDER BY created_at DESC LIMIT 1"` — expect non-null GPS.
   - Download the R2 object from the response URL. Run `exiftool <file>` or `identify -verbose <file>` — expect no GPS tags, no APPn segments other than what the Worker itself re-injects (if any).
