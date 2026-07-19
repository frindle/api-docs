# API Documentation

Notes on third-party APIs used across projects.

## Sources

- [Amazon](Amazon/api.md) — DOM scraping + background fetch worker
- [BFMR](BFMR/api.md) — resale deals platform (REST, API-KEY/API-SECRET)
- [BigSkyBuyers](BigSkyBuyers/api.md) — tRPC, cookie-based auth
- [BuyingGroup](BuyingGroup/api.md) — buying group platform (Django SimpleJWT, POST-based)
- [CardCenter](CardCenter/api.md) — gift card resale platform (Bearer token)
- [Costco](Costco/api.md) — GraphQL, MSAL Bearer token
- [Pushover](Pushover/api.md) — push notifications (token+user in body; priority levels; free-tier limits)
- [Reolink](Reolink/api.md) — camera/NVR HTTP API (token auth, Search/Playback/Download for recorded clips, Range-seekable playback)
- [Rivian](Rivian/api.md) — unofficial GraphQL gateway (CSRF + session tokens, ~90-day sessions, opaque throttling)
- [SolarEdge](SolarEdge/api.md) — inverter Modbus/TCP (port 1502)
- [Tesla](Tesla/api.md) — Fleet API (OAuth scopes, energy sites, wall connectors, telemetry, command proxy)
- [Walmart](Walmart/api.md) — React SPA DOM scraping + `__NEXT_DATA__`
