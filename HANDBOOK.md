# Operator handbook

CleanLoop is the Northbridge collection control room. This handbook covers the daily operating routine, the decisions each screen supports, and the safe limits for thresholds and scheduling.

## 1. Start of shift (07:00)

1. Open **Overview**.
2. Confirm the city pulse and that “On the road” matches the crews you expect.
3. Check **Signal watch** for critical bins, offline sensors, fleet exceptions and delayed routes.
4. Open **Live operations** and confirm each active route has a driver, vehicle and next stop.

If a route is missing or the board is empty, go to **Schedule** and check the day agenda before creating anything new.

## 2. Planning tomorrow

Open **Route planner**.

| Control | What it does | Safe range |
| --- | --- | --- |
| Collection date | Day being planned | Tomorrow or later |
| Fill threshold | Projected fill at which a bin is collected | 60–75% |
| Max bin age | Forces collection after this many days | 7–14 days |
| Objective | Balances cost, emissions, service or a mix | `balanced` for normal days |
| Fleet | Road-ready vehicles available to the plan | Leave all selected unless a vehicle is unavailable |

Recommended routine:

1. Keep the saved settings threshold unless there is a specific reason to change it.
2. Review the forecast queue. Priority and high-fill bins should be at the top.
3. Deselect vehicles that are charging or needed elsewhere.
4. Generate the plan. Read the result strip: routes, stops, distance saved, CO₂e avoided and any skipped bins.
5. Open each route and check the stop order and ETAs against local knowledge.
6. Start a route only when the crew is actually departing.

### Choosing an objective

- **Balanced** — sensible default; keeps crew availability and payload in view.
- **Lowest cost** — favours fewer, larger routes. Watch route duration.
- **Lowest emissions** — puts electric and hybrid vehicles first. Confirm range is sufficient.
- **Fastest service** — adds capacity when the day is heavy. Useful after a missed collection day.

### If bins are skipped

The result lists each skipped bin with a reason. Common causes:

- **No remaining vehicle capacity today** — add a crew or lower the threshold pressure by planning earlier.
- **No vehicle available for route** — free a vehicle or reduce the selected fleet.
- **Load exceeds vehicle capacity** — the group is too heavy for the assigned vehicle; regenerate after changing the fleet.

Do not delete bins to make a plan fit. Adjust fleet or threshold instead.

## 3. Auto-planning the week

Open **Schedule → Auto-plan week**.

1. Choose a start date no earlier than tomorrow.
2. Pick a 7, 10 or 14 day window.
3. Set the forecast threshold. Match the operational setting unless you are deliberately planning conservatively.
4. Review the result banner: routes generated, days covered, bins scheduled, distance saved and deferred bins.

The planner merges very small days into neighbouring runs so crews are not sent out for one or two bins. It also **replaces unstarted scheduled routes** inside the window. Active and completed routes are protected, and their bins are excluded.

### Week review

Use the calendar to spot:

- Days with no plan and open capacity.
- Days carrying more routes than crews.
- Weekend load that could be consolidated.
- A day with only one route that may be merged with the day before or after.

The day agenda shows the full route cards, crew coverage and the day summary. Use **Track** to follow an active route.

## 4. Running the day

**Live operations** refreshes every 20 seconds while auto-refresh is on.

- The map shows every route, all bins and the selected route’s stop order.
- **Route focus** shows the driver, vehicle, next stop, fill level, service time and remaining work.
- **Log this collection** completes the next stop, records the weight and updates the bin.
- The stop timeline marks completed stops and highlights the next one.
- **Signal watch** surfaces exceptions that may affect service.

Log stops as they happen. Delayed logging makes the board, analytics and next-day forecast less accurate.

### Handling exceptions

| Signal | First action |
| --- | --- |
| Critical bin (85%+) | Check whether it is already on an active route; if not, request priority pickup or plan it |
| Offline sensor | Treat the bin as unknown; inspect manually before relying on fill |
| Vehicle in service | Reassign the route in the planner or move the vehicle to available when repaired |
| Route delayed | Check the stop timeline; advance completed stops or re-plan the remainder |
| No remaining stops | The route should complete automatically after the last stop |

## 5. Bins

Open **Smart bins**.

- Search by bin name, street or ID.
- Filter by zone, waste stream and status.
- Select a bin to inspect fill, sensor health, access notes and recent collections.
- **Mark collected** logs a collection and resets fill.
- **Request pickup** flags the bin as priority, guaranteeing inclusion in the next plan.
- **Add bin** registers a new asset. New bins start with the fill and rate you enter.

Keep access notes short and useful for crews: gate codes, alley width, parking restrictions.

## 6. Fleet and crews

Open **Fleet**.

- Change status to **Available**, **Charging** or **In service** as the depot changes.
- A vehicle on an active route cannot be changed until the route completes.
- **Add vehicle** registers a new asset and links it to a driver.
- The driver roster shows licence, shift, assigned vehicle, rating and contact.

Charging and maintenance vehicles are excluded from new plans, so keep statuses current.

## 7. Analytics

Open **Analytics** and choose 7, 14 or 30 days.

- **Collection volume** — daily tonnage by stream; toggle streams to isolate recycling or organics.
- **Waste composition** — share of collected tonnage and diversion rate.
- **Zone performance** — tonnage, average fill, on-time percentage, diversion and cost per tonne.
- **Demand by weekday** — use this to anticipate heavy days.
- **Efficiency pulse** — fleet utilisation, on-time and diversion.
- **Optimized vs baseline** — distance saved per route against a fixed stop order.

Use the range switcher before drawing conclusions; a single day is noisy.

## 8. Settings

Open **Settings** to change the operating policy. Every page and optimizer run uses these values.

| Setting | Effect | Recommended |
| --- | --- | --- |
| Collection threshold | When a projected bin becomes due | 60–70% |
| Max age between collections | Forces service even at low fill | 10 days |
| Optimization objective | Default scoring for plans | Balanced |
| Shift start/end | Service window used for ETAs | 06:30–16:30 |
| Average travel speed | ETA and duration estimate | 24–30 km/h |
| Distance cost, labour, fuel | Cost and savings estimates | Match current contract rates |
| Diesel emissions factor | CO₂e conversion | 2.68 kg/L unless your fleet differs |
| Notifications | Which signals reach dispatch | Keep critical bins and offline sensors on |

Changing settings does **not** rewrite existing routes. Regenerate a plan to apply a new threshold or objective.

## 9. Reset and recovery

### Reset the demo

**Settings → Reset demo data** or:

```bash
npm run reset:data
```

A reset restores 36 bins, 8 vehicles, 12 routes and 60 days of analytics. Settings, zones, depots and drivers are preserved. New bins, status changes and completed stops are cleared.

If the API is running, use the Settings reset button or `POST /api/admin/reset` so the server cache is refreshed. The CLI command alone updates the file; restart the server afterwards.

### Common issues

| Symptom | Check |
| --- | --- |
| Pages show “We couldn’t load this view” | Is the API running on port 4000? Open `/api/health` |
| Data looks stale after a reset | Restart the server or use the in-app reset |
| Plan generates but no routes | Threshold too high, no eligible bins, or no road-ready vehicles |
| Route missing from the day | Filters on Schedule, or the date was replaced by a new plan |
| Vehicle cannot change status | It is assigned to an active route |
| Bin cannot be deleted | It is on an active or planned route |

## 10. Guardrails

- Do not plan more than six crews per day unless the fleet genuinely supports it.
- Keep a threshold below 85%; collecting at very high fill risks overflow and missed service.
- Do not defer bins beyond the max-age limit for more than a day.
- Review skipped bins before starting routes.
- Re-plan rather than manually deleting stops; the optimizer preserves capacity guarantees.
- Reset only the local demo store; never point these scripts at production data.
