# Optimization guide

CleanLoop uses a transparent, deterministic pipeline. The same bins, fleet and settings always produce the same plan, which makes the demo easy to explain, test and replay.

```
Forecast demand → Select fleet → Sweep clusters → Rebalance load → Sequence stops → Score plan
```

## 1. Forecast demand

Each bin is projected forward to the requested collection date:

```
projected fill = current fill + fill rate per day × days ahead
```

A bin is **eligible** when any of these is true:

- projected fill ≥ collection threshold
- the bin is flagged `priority = high`
- the days since the last collection ≥ `maxAgeDays`

Eligibility also produces an urgency score:

```
urgency = projected fill
        + 18 if priority
        + min(12, max(0, age − 3) × 1.5)
```

Bins are sorted by urgency. Bins already protected by an active or completed route are excluded.

## 2. Select the fleet

Only road-ready vehicles are considered: maintenance vehicles are excluded, and the caller can restrict the set with `vehicleIds`. Vehicles are ranked by objective:

| Objective | Ranking |
| --- | --- |
| `balanced` (default) | Available first, then largest payload |
| `cost` | Largest payload first, then drivetrain |
| `emissions` | Electric → hybrid → diesel, then payload |
| `service` | Largest payload, with an extra crew when the workload is large |

The optimizer then estimates the number of crews needed:

1. Add the largest-ranked vehicles until usable capacity (88% of payload) covers the total bin weight.
2. Ensure at least `ceil(bins ÷ 14)` crews, because a route is capped at **14 stops**.
3. Cap the plan at six crews.
4. Add a crew if capacity is still short, or if `service` is selected and the plan has more than nine stops.

## 3. Cluster by geography (angular sweep)

Bins are converted to polar coordinates around the depot. The sweep:

1. Sorts every bin by angle.
2. Finds the largest angular gap and starts the ring there, avoiding an artificial cut through a dense neighbourhood.
3. Walks the ring, closing a cluster when it has accumulated roughly its share of total weight (90% of target) and enough bins remain to fill the remaining crews.
4. Merges any leftover clusters so the number of groups never exceeds the number of crews.

This is a classic sweep heuristic. It produces compact, contiguous service areas without requiring an external solver.

## 4. Rebalance capacity

Groups are paired with vehicles largest-first. If a group exceeds 92% of its vehicle's payload, the lowest-fill non-priority bins are moved to another group that has room. If no group can accept the bin, it is skipped and reported with a reason such as `No remaining vehicle capacity today`.

Priority bins are never dropped by rebalancing.

## 5. Sequence stops (nearest neighbour + 2-opt)

Each cluster is sequenced independently:

1. **Nearest neighbour** — start at the depot and repeatedly visit the closest unvisited bin.
2. **2-opt** — test reversing every sub-segment. If the reversal shortens the route, keep it and repeat. The loop is capped at 12 passes; routes with fewer than four stops skip this step.

Distance uses city-map units scaled to kilometres with a road factor:

```
km = hypot(Δx, Δy) × 0.145 × 1.16
```

Every route starts and ends at the depot of the plan.

## 6. Stop timing

Each stop receives an ETA:

```
travel minutes  = leg km ÷ travel speed × 60
service minutes = round((1.4 + litres ÷ 260) × 2) ÷ 2
```

The route ends after the last service plus the return leg. Driver shift start is used when available, otherwise the configured shift start.

## 7. Load, emissions and baseline

Tonnage is estimated from capacity, fill and waste density:

| Stream | Density (kg/L) |
| --- | --- |
| General | 0.24 |
| Recycling | 0.09 |
| Organic | 0.34 |
| Glass | 0.46 |

The **baseline** is deliberately simple: the same bins visited in bin-ID order. Savings are the difference between that fixed order and the optimized sequence:

```
fuel saved (L) = saved km × 0.29
CO₂e saved     = fuel saved × diesel CO₂ factor
cost saved     = saved km × distance cost
```

Route cost combines distance and labour:

```
cost = distance × cost per km + (duration ÷ 60) × labour rate
```

These figures are estimates for planning, not audited carbon accounting.

## 8. Multi-day auto-planning

`POST /api/schedule/auto` applies the same thinking across a date window:

1. For each eligible bin, find its **earliest due date** inside the window.
2. Group bins by that date.
3. Merge groups smaller than three bins into the previous day, or into the last day if the first group is small. This avoids sending a crew out for one or two bins.
4. Run the optimizer for each merged day with the same threshold and the available fleet.
5. Replace unstarted scheduled routes in the window. Active and completed routes are protected, and their bins are excluded from the new plan.

## Determinism and limits

- Ties are broken by bin or vehicle ID.
- No randomness is used anywhere in the pipeline.
- A maximum of 14 stops per route keeps service times realistic.
- Plans are capped at six crews per day.
- Capacity is a hard constraint; overflow is reported rather than silently dropped.

## Extension points

The pipeline is intentionally modular. To productionise it:

- Replace map-unit distance with a road-network matrix (OSRM, OpenRouteService or Google Routes).
- Add time windows per bin or customer.
- Swap the sweep + 2-opt sequencer for OR-Tools or VROOM while keeping the same route payload.
- Feed real scale data and calibration factors into tonnage and emissions estimates.
- Add driver shift rules, break enforcement and depot reloads for longer routes.
