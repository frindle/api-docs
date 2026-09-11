# AeroDataBox (via RapidAPI)

Used by `flight-staffing-forecast` for future-date Allegiant (G4) schedules.

## Quota / pricing — measured live 2026-08-01

Free tier headers on every response:
- `X-RateLimit-API-Units-Limit: 600` (per month)
- `X-RateLimit-Requests-Limit: 2400`

**The airport-flights endpoint costs a flat 2 API units per call.** Measured
by diffing `X-RateLimit-API-Units-Remaining` across calls:
- Independent of window length — a 2-hour window costs the same as 12h.
- Independent of `direction` — `Both` costs the same as `Departure`.
- A `204 No Content` response is still billed 2 units.
- A `429` is not billed.

Window is capped at 12h, so **a full local day = 2 calls = 4 units per airport**.

## `direction=Both` is strictly better than `Departure`

Same cost, roughly double the usable coverage. With `withLeg=true`, an
*arrival* item carries the origin airport plus that flight's scheduled
departure time in origin-local:

```json
{"departure": {"airport": {"iata": "FNT", ...},
               "scheduledTime": {"local": "2026-08-03 07:00-04:00"}},
 "arrival":   {"scheduledTime": {"local": "2026-08-03 08:12-07:00"}},
 "number": "G4 3150"}
```

So one `Both` query at an airport yields that airport's own departures *and*
the departures of every spoke feeding it. Verified at KLAS: 29 outbound + 30
inbound legs, inbound origins spanning 28 stations never queried.

The queried airport is only named on the *opposite* side of the leg — fill it
in from the query itself.

## No airline-wide schedule endpoint

Confirmed: lookups are per-airport or per-flight-number only. There is no
"all G4 flights everywhere on date X" call. Matching a system-wide scope means
one query per station, which is why base selection matters (see
Projects/flight-staffing-forecast).

## Endpoint shape

```
GET /flights/airports/icao/{icao}/{fromLocal}/{toLocal}
  ?direction=Both&withLeg=true&withCancelled=true&withCodeshared=true
  &withCargo=false&withPrivate=false&withLocation=false
Headers: x-rapidapi-host: aerodatabox.p.rapidapi.com, x-rapidapi-key: ...
```

Local datetimes are ISO with no offset (`2026-08-03T00:00`). Cancellations
come through `status` (`Canceled`, `CanceledUncertain`), not a boolean.
Schedule density thins several days out — a LAS morning 4 days ahead returned
a single G4 leg where the same window 2 days ahead returned 14.

## Pricing tiers (RapidAPI, confirmed 2026-08-06)

| Tier | Cost | Units/month |
|---|---|---|
| Free | $0 | 600 |
| Pro | **$5.35/mo** | 6,000 |
| Ultra | $32/mo | 60,000 |
| Mega | $160/mo | 600,000 |

Full G4 system coverage (22 derived operating bases, ~95.1% of flights) at
the corrected cadence (see flight-staffing-forecast project note — a
`SCHEDULE_TTL_SEC`/`SCHEDULE_HORIZON_DAYS` mismatch bug was inflating pulls
from 2/date to 3/date) costs **5,456 units/month** — fits inside the Pro
tier with headroom, does not fit free. Tier 2 (airport-flights) cost is flat
2 units/call regardless of window size or `direction` — confirmed again via
3 controlled live deltas 2026-08-06, matches the 2026-08-01 finding above.
