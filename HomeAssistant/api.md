# Home Assistant

Local instance at <lan-ip>:8123 (see homelab/devices.md). Two separate
integration surfaces in use: the MCP server (for Claude Code) and the plain
REST/webhook API (used by other services, e.g. ev-dashboard).

## MCP Server (for Claude Code)
- Server name: `home-assistant`, added at **user** scope (works in any
  project): `claude mcp add --scope user --transport sse home-assistant
  http://<lan-ip>:8123/mcp_server/sse --header "Authorization: Bearer
  <token>"`.
- **Must use `--transport sse` and the `/mcp_server/sse` path** — this
  instance's MCP Server integration only supports the older SSE transport.
  `/mcp_server/mcp` (streamable-HTTP) 404s.
- Auth token is a real HA **Long-Lived Access Token** (JWT, three
  dot-separated base64 segments) from the user's HA profile page →
  "Long-Lived Access Tokens" → Create Token. A short ULID-looking string is
  NOT valid — confirmed by testing directly against `/api/`, got 401.
- Newly-added/reconnected MCP servers don't always show up mid-session —
  `claude mcp get home-assistant` can report "Connected" while ToolSearch
  still finds nothing until a fresh session starts.
- Standing instruction: prefer these HA tools (state, control, sensors,
  automations) over raw LAN calls or asking the user to do it manually,
  when a task is a good fit.

## Webhooks / REST (used by other services)
- ev-dashboard's Rivian-arrival garage-lights automation calls
  `http://<lan-ip>:8123/api/webhook/<id>` directly over LAN — no
  Cloudflare involved, HA has no Access app in front of it.
- `homeassistant.local` (mDNS) does **not** resolve from inside Docker
  containers (no multicast DNS there) — always use the static IP
  (<lan-ip>) for container-to-HA calls.

## Gotcha: `condition: sun` midnight discontinuity
The built-in `condition: sun` with `after: sunset` compares against the
*next occurring* sunset/sunrise event, which flips to the next calendar
day's event right after local midnight — so `after: sunset` silently
evaluates **false** from midnight until that evening's sunset, even though
it's clearly still night. Caused a real missed automation trigger at
12:43 AM. Fix: a template condition explicitly ORing "within N hours of the
upcoming sunset" with "before the upcoming sunrise"
(`state_attr('sun.sun','next_setting')`/`next_rising`), which has no
midnight discontinuity. Check any other automation using `condition: sun`
with an offset for the same bug.

## Integrations of note
- **LocalTuya** (HACS custom integration) — see Tuya.md. Cloud API account
  entry exists but zero devices configured as of 2026-07-26.
- **Tuya** (official) — 2 devices, unrelated to the Westinghouse/LocalTuya
  effort (not yet inventoried which devices).
