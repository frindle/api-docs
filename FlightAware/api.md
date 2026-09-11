# FlightAware AeroAPI

Evaluated 2026-08-06 as an alternative to AeroDataBox for
`flight-staffing-forecast`'s live 24-48hr-ahead schedule data. User chose
this direction over AeroDataBox's paid tier.

## Pricing (confirmed 2026-08-06)

Personal tier: **$5/month usage credit**, no monthly minimum (**$10/month
credit** if you host an ADS-B feeder — needs a ~$20 SDR dongle on a spare
Pi). Billed **$0.020 per result set**, a result set = up to **15 records**.
Rate limit: 10 result sets/min on Personal.

## Schedule endpoint

`GET /schedules/{start}/{end}` — supports `airline=`, `origin=`,
`destination=`, `flight_number=` filters. Published airline schedules span
**3 months in the past to 1 year in the future**. Critically, **one query
with `airline=G4` returns the whole network for a date** — unlike
AeroDataBox, which is per-airport only and has no airline-wide endpoint.
This also closes the "inside turn" coverage gap AeroDataBox couldn't reach
(mid-rotation departures from non-base stations).

**Unverified as of this note**: whether `/schedules` is actually available
on the Personal tier vs gated to a higher tier — a community forum thread
only confirms `/alerts` and `/history/*` are Personal-tier-accessible.
Confirm at signup before building against it.

**Correction, 2026-08-08**: the "unverified" note above had the tier-gating
backwards. Independently confirmed via `github.com/chrischall/flightaware-mcp`'s
`docs/FLIGHTAWARE-API.md` — a real live-verification log dated 2026-06-23
against an actual Personal-tier key: `fa_get_scheduled_flights`
(`GET /schedules/{start}/{end}`) returned a real `200` with `{scheduled[],
...}`. `/alerts*` and `/history/*` DID 401 on that same Personal-tier key —
tier-gated, matching FlightAware's own error body ("Alerts and Historical
data are only available on Standard and Premium tiers"). So `/schedules`
access is NOT actually in doubt the way this note previously implied — the
earlier forum-thread source apparently had it backwards, or was
misremembered when this note was written. Not FlightAware's own official
confirmation, so still worth a sanity check at real signup, but this is
strong evidence, not a coin flip.

**Also confirmed** (same source + FlightAware's public OpenAPI spec,
`aeroapi-openapi.yml`): auth is header `x-apikey: <key>`, NOT
`Authorization: Bearer`. `scheduled_out`/`scheduled_in` in the response
are **UTC**, not station-local like AeroDataBox's `scheduledTime.local` —
a real difference this project's `app/sources/flightaware.py` (built
2026-08-08) handles via `app/airports.py`'s existing `zone_for()` IATA→IANA
table, converting to match AeroDataBox's local-time contract exactly so
forecast.py doesn't need to know which source a leg came from. `/schedules`
has no live cancelled/delayed signal (published-schedule data only, per
FlightAware's own spec description) — `status`/`cancelled` are placeholder
constants in the client until a live-status source gets added alongside it.

## Cost math for full G4 network coverage

At ~398 flights/day (BTS 2026-03 sample, 12,323 flights/31 days), whole-
network 1-pull/date: ~27 result sets/day → ~837/month → **~$16.74/month
gross**, netting to **~$9-12/month** against the $5 credit, or
**~$4-7/month** with the ADS-B feeder's $10 credit. Cheaper than
AeroDataBox's Pro tier ($5.35/mo for only 95.1% coverage) while reaching
100% and closing the inside-turn gap.

## Free-tier combination math (for reference, not the chosen path)

Free AeroDataBox (4 bases, the max the 600-unit budget affords at the
corrected 1-pull/date cadence) + FlightAware's $5 free credit spent on 2-3
more high-volume bases (AZA, SRQ, VPS) = **76.8%** coverage — a real
improvement over AeroDataBox-only free (52%), but nowhere near 95-100%,
and the FlightAware free credit is usage-based with no hard cap (a heavy
day bills over, no warning). Not chosen — see project note.

## Other free sources checked, none viable

- **aviationstack**: free tier's `flight_date` param returns
  `function_access_restricted` — confirmed live 2026-07-26, scheduled data
  needs Basic+ paid tier.
- **OpenSky Network**: live ADS-B state vectors only, no schedule concept
  at all.
- **FAA SWIM/SCDS (SFDPS)**: requires a signed Service Access Agreement +
  Solace JMS client, and even then it's real-time en-route NAS data, not
  published airline schedules — wrong data type regardless of access
  friction.
- **Aviation Edge**: no permanent free tier, pricing page listed a
  suspicious $299 first month — low confidence, not pursued.
