# API Documentation

Notes on third-party APIs used across projects.

## Sources

- [AeroDataBox](AeroDataBox/api.md) — RapidAPI flight schedules (per-airport, API-unit metered; G4 forecasting)
- [Amazon](Amazon/api.md) — DOM scraping + background fetch worker; iris-iframe card last-4
- [BFMR](BFMR/api.md) — resale deals platform (REST, API-KEY/API-SECRET)
- [BigSkyBuyers](BigSkyBuyers/api.md) — tRPC, cookie-based auth
- [BuyingGroup](BuyingGroup/api.md) — buying group platform (Django SimpleJWT, POST-based)
- [CardCenter](CardCenter/api.md) — gift card resale platform (Bearer token)
- [Costco](Costco/api.md) — GraphQL, MSAL Bearer token
- [Darkbloom](Darkbloom/api.md) — local inference provider (OpenAI-compatible endpoint, model loading/serving behavior)
- [Deluge](Deluge/api.md) — torrent JSON-RPC / WebUI (move_storage/recheck semantics, container path topology)
- [Fi](Fi/api.md) — GPS dog collar; unofficial GraphQL (email/password), collar joins WiFi natively (no Base needed), pytryfi client; no official API
- [Firewalla](Firewalla/api.md) — router/firewall, local + consumer cloud + MSP APIs
- [FlightAware](FlightAware/api.md) — AeroAPI flight schedules (airline-wide endpoint, `x-apikey`, usage-credit pricing)
- [HomeAssistant](HomeAssistant/api.md) — local MCP server (SSE) + REST/webhook API
- [Pushover](Pushover/api.md) — push notifications (token+user in body; priority levels; free-tier limits)
- [Reolink](Reolink/api.md) — camera/NVR HTTP API (token auth, Search/Playback/Download for recorded clips, Range-seekable playback)
- [Rivian](Rivian/api.md) — unofficial GraphQL gateway (CSRF + session tokens, ~90-day sessions, opaque throttling)
- [SolarEdge](SolarEdge/api.md) — inverter Modbus/TCP (port 1502)
- [Teams](Teams/api.md) — MS Teams Shifts via captured first-line-worker token (`flw.teams.cloud.microsoft`, NOT public Graph); ICS/staffing export
- [Tesla](Tesla/api.md) — Fleet API (OAuth scopes, energy sites, wall connectors, telemetry, command proxy)
- [Tuya](Tuya/api.md) — local LAN protocol for whitelabeled Tuya devices (no cloud dependency once local_key is known); includes Westinghouse WPAC12000S DP map
- [Walmart](Walmart/api.md) — React SPA DOM scraping + `__NEXT_DATA__`
