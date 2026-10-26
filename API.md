# CleanLoop API reference

All endpoints are served by the local Express service.

- Base URL: `http://localhost:4000/api`
- Content type: `application/json`
- Errors: JSON body `{ "error": "message" }` with an appropriate HTTP status
- No authentication is required for the local demo

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/health` | Service status and dataset counts |
| GET | `/meta` | City, zones, depots, drivers and objectives |
| GET | `/dashboard` | Overview payload: KPIs, routes, alerts, activity |
| GET | `/bins` | Smart bins with filters and summary |
| GET | `/bins/:id` | One bin plus recent collection history |
| POST | `/bins` | Register a smart bin |
| PATCH | `/bins/:id` | Update bin fields |
| POST | `/bins/:id/request-pickup` | Flag a bin for priority collection |
| DELETE | `/bins/:id` | Remove an unassigned bin |
| GET | `/vehicles` | Fleet, drivers and current/next route |
| POST | `/vehicles` | Add a vehicle |
| PATCH | `/vehicles/:id` | Update vehicle fields or status |
| DELETE | `/vehicles/:id` | Remove an unassigned vehicle |
| GET | `/drivers` | Driver roster with assigned vehicle |
| GET | `/routes` | Routes filtered by date, window or status |
| GET | `/routes/:id` | One route with recent activity |
| POST | `/routes/optimize` | Generate and persist optimized routes |
| PATCH | `/routes/:id/status` | Set scheduled, active, completed, delayed or cancelled |
| POST | `/routes/:id/advance` | Mark the next stop collected |
| GET | `/schedule` | Seven-day (configurable) schedule payload |
| POST | `/schedule/auto` | Forecast and auto-plan a date window |
| GET | `/analytics` | 7, 14 or 30 days of operational analytics |
| GET | `/priority-bins` | Forecast bins needing collection soon |
| GET | `/settings` | Operational settings |
| PUT | `/settings` | Update operational settings |
| GET | `/resources` | Dataset counts, downloads, API list, docs, reading |
| GET | `/export/bins.csv` | Bin dataset as CSV |
| GET | `/export/vehicles.csv` | Fleet register as CSV |
| GET | `/export/routes.csv` | Route plan as CSV |
| GET | `/export/bins.geojson` | Bin points as GeoJSON |
| GET | `/export/dataset.json` | Complete dataset snapshot |
| POST | `/admin/reset` | Restore the seeded demo dataset |

---

## Health and metadata

### `GET /health`

```json
{
  "status": "ok",
  "service": "cleanloop-api",
  "version": "1.0.0",
  "city": "Northbridge",
  "dataStore": "db.json",
  "routes": 12,
  "bins": 36
}
```

### `GET /meta`

Returns `city`, `country`, `region`, `zones`, `depots`, `drivers`, `wasteTypes` and `objectives`.

---

## Bins

### `GET /bins`

Query parameters (all optional): `q`, `zone`, `type`, `status`.

```json
{
  "bins": [
    {
      "id": "BIN-001",
      "name": "Cathedral Square",
      "address": "2 Minster Yard",
      "zoneId": "ZN-01",
      "zone": { "id": "ZN-01", "name": "Cathedral Quarter", "color": "#2F7A5D" },
      "type": "general",
      "capacityLiters": 1100,
      "fillPct": 72,
      "fillRatePerDay": 1.8,
      "status": "attention",
      "priority": "normal",
      "ageDays": 6,
      "daysUntilFull": 15,
      "nextCollection": { "date": "2026-10-27", "source": "forecast", "eta": null },
      "sensorStatus": "online",
      "sensorBattery": 87
    }
  ],
  "total": 36,
  "allCount": 36,
  "summary": {
    "total": 36,
    "critical": 2,
    "attention": 7,
    "offline": 1,
    "normal": 26,
    "avgFill": 46.4
  }
}
```

### `POST /bins`

Required: `name`, `address`, `zoneId`, `type`.

Optional: `capacityLiters` (120–5000), `fillPct` (0–100), `fillRatePerDay` (0–25), `sensorStatus` (`online` / `offline`), `sensorBattery` (0–100), `priority` (`normal` / `high`), `accessNotes`, `lastCollected`, `x`, `y`.

Returns `201` with `{ bin, message }`.

### `PATCH /bins/:id`

Accepts the same fields as create, except `id`. Returns `{ bin, message }`.

### `POST /bins/:id/request-pickup`

Sets `priority` to `high` so the bin is included in the next optimized plan. Returns `{ bin, message }`.

---

## Vehicles and drivers

### `GET /vehicles`

Each vehicle includes `driver`, `currentRoute` and `nextRoute`. A `summary` object reports totals by status and drivetrain.

### `POST /vehicles`

Required: `name`, `model`, `plate`, `driverId`. Optional: `type` (`electric` / `hybrid` / `diesel`), `capacityKg` (1000–20000), `rangeKm`, `body`.

New vehicles start as `available`.

### `PATCH /vehicles/:id`

Updatable: `name`, `model`, `plate`, `type`, `capacityKg`, `status`, `driverId`, `fuelPct`, `rangeKm`, `odometerKm`, `nextService`, `body`.

Valid statuses: `available`, `on-route`, `charging`, `maintenance`. A vehicle assigned to an active route cannot be returned to `available` until the route ends.

### `GET /drivers`

Returns each driver with the vehicle currently linked through `driverId`.

---

## Routes

### `GET /routes`

Optional query parameters:

- `date` — exact ISO date
- `start`, `end` — inclusive ISO date window
- `status` — comma-separated list, for example `active,completed`

Response: `{ routes, count, summary }`. Every route includes stops with bin, ETA, weight, completion state, plus distance, load, progress, cost and emissions metrics.

### `POST /routes/optimize`

Generate a plan and persist it.

```json
{
  "date": "2026-10-27",
  "threshold": 65,
  "objective": "balanced",
  "vehicleIds": ["VH-01", "VH-02", "VH-04"],
  "depotId": "DEP-01",
  "replace": true
}
```

- `date` defaults to tomorrow.
- `threshold` is the projected fill percentage at which a bin becomes eligible.
- `objective` is `balanced`, `cost`, `emissions` or `service`.
- `vehicleIds` restricts which road-ready vehicles may be used.
- `replace` (default `true`) removes **unstarted scheduled** routes for that date before inserting the new plan. Active and completed routes and their bins are protected.

Response:

```json
{
  "date": "2026-10-27",
  "objective": "balanced",
  "threshold": 65,
  "stats": {
    "routes": 3,
    "stops": 27,
    "distanceKm": 84.2,
    "baselineKm": 116.5,
    "savedKm": 32.3,
    "savedPct": 27.7,
    "weightKg": 9120,
    "co2SavedKg": 22.4,
    "costSaved": 69.4
  },
  "skipped": [{ "binId": "BIN-014", "reason": "No remaining capacity" }],
  "routes": [],
  "message": "3 optimized routes scheduled for 2026-10-27"
}
```

### `PATCH /routes/:id/status`

```json
{ "status": "active" }
```

Setting `completed` marks every stop complete, records weights and resets collected bins. Setting `active` updates vehicle availability.

### `POST /routes/:id/advance`

Marks the next uncollected stop as complete, logs a collection and updates the bin fill. Repeating this on the last stop completes the route and releases the vehicle.

```json
{ "route": {}, "message": "Stop logged at BIN-008" }
```

---

## Schedule

### `GET /schedule?start=2026-10-26&days=7`

`start` defaults to the current week's Monday. `days` is clamped to 1–14. The payload contains `startDate`, `endDate`, `days[]` (each with weekday, day number, routes and summary), `summary`, `fleet` and `crews`.

### `POST /schedule/auto`

```json
{
  "startDate": "2026-10-27",
  "days": 7,
  "threshold": 65
}
```

Forecasts demand, merges low-demand days into shared crew runs, replaces unstarted scheduled routes in the window, and protects active/completed collections.

Response includes `generated`, `daysWithRoutes`, `scheduledBins`, `skippedBins`, `savings` (`distanceKm`, `co2Kg`, `cost`), `generatedRoutes`, `week` and `message`.

---

## Analytics

### `GET /analytics?range=30`

`range` accepts `7`, `14` or `30`; anything else falls back to 30.

Payload includes:

- `series[]` — daily tonnage by stream, distance, fuel, cost, routes, stops, on-time and diversion rates
- `composition[]` — tonnage share by waste type
- `weekdayPattern[]` — average tonnage by weekday
- `zones[]` — bins, tonnage, trend, average fill, on-time, diversion, cost per tonne
- `routeComparison[]` — optimized distance versus baseline stop order
- `totals`, `previousTotals`, `deltas`, `fleet`, `startDate`, `endDate`

---

## Settings

### `PUT /settings`

Any subset of:

```json
{
  "city": "Northbridge",
  "country": "United Kingdom",
  "region": "West Mercia",
  "depotId": "DEP-01",
  "collectionThreshold": 65,
  "maxAgeDays": 10,
  "shiftStart": "06:30",
  "shiftEnd": "16:30",
  "travelSpeedKph": 26,
  "costPerKm": 2.15,
  "laborPerHour": 31,
  "fuelPrice": 1.72,
  "dieselCo2KgPerL": 2.68,
  "costPerTonne": 88,
  "optimizationObjective": "balanced",
  "autoScheduleDays": 7,
  "notifications": {
    "criticalBin": true,
    "sensorOffline": true,
    "routeDelay": true,
    "weeklyDigest": true
  }
}
```

Notifications are merged with the existing object. Threshold is clamped to 10–100; travel speed to 5–80; distance cost to 0–50.

---

## Exports

| Path | Format |
| --- | --- |
| `/api/export/bins.csv` | CSV, one row per bin |
| `/api/export/vehicles.csv` | CSV, one row per vehicle |
| `/api/export/routes.csv` | CSV, one row per stop |
| `/api/export/bins.geojson` | GeoJSON FeatureCollection |
| `/api/export/dataset.json` | Full snapshot with settings, bins, vehicles, routes, analytics |

The complete dataset includes `meta.generatedAt`, `meta.city` and `meta.version`.

---

## Reset

### `POST /api/admin/reset`

Rebuilds the seed dataset, clears the in-memory cache and returns the restored settings plus counts. This is the same operation as `npm run reset:data`, but safe to call while the server is running.

---

## Error codes

| Status | Meaning |
| --- | --- |
| 400 | Missing or invalid field, unknown zone, driver, type or objective |
| 404 | Bin, vehicle or route not found |
| 409 | Conflict: deleting a bin/vehicle assigned to a route, or advancing a completed route |
| 500 | Unexpected server error (body includes a message) |

## cURL examples

```bash
curl http://localhost:4000/api/health

curl -X POST http://localhost:4000/api/routes/optimize \
  -H "Content-Type: application/json" \
  -d '{"date":"2026-10-27","threshold":65,"objective":"emissions"}'

curl -X POST http://localhost:4000/api/schedule/auto \
  -H "Content-Type: application/json" \
  -d '{"startDate":"2026-10-27","days":7,"threshold":65}'

curl -X POST http://localhost:4000/api/routes/RT-3181/advance
```
