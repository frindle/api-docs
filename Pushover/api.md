# Pushover

Push notification API used across the homelab projects (ev-dashboard,
plex-automation, award-search, teams-shifts-docker, network-bandwidth-monitor).

- **Docs:** https://pushover.net/api
- **Endpoint:** `POST https://api.pushover.net/1/messages.json`
- **Auth:** app token + user key in the request body (no headers).
  Each project registers its own app at https://pushover.net/api so pushes
  are attributable and rate limits are per-app.

## Request

JSON or form-encoded both work:

```json
{
  "token":   "<app token>",
  "user":    "<user key>",
  "title":   "EV Dashboard — Tesla re-auth needed",
  "message": "Body text, up to 1024 chars",
  "priority": 0,
  "url":      "https://optional-supplementary-link",
  "url_title": "Open queue"
}
```

### Priority levels
| Value | Behavior |
|-------|----------|
| -2 | No notification, badge only |
| -1 | Quiet — no sound/vibration |
| 0  | Normal (default) |
| 1  | High — bypasses quiet hours |
| 2  | Emergency — repeats until acknowledged; **requires** `retry` (≥30s) and `expire` (≤10800s) params |

## Response
- `200` with `{"status": 1, "request": "<uuid>"}` on success.
- `4xx` with `{"status": 0, "errors": ["..."]}` — invalid token/user is a 400,
  not a 401.

## Limits (confirmed)
- 10,000 messages/app/month on the free tier; resets on the 1st.
- Message ≤ 1024 chars, title ≤ 250 chars, URL ≤ 512 chars.
- No documented per-minute rate limit at homelab volumes; treat 5xx as
  transient and drop the message rather than retry-looping (all our
  integrations are fire-and-forget).

## Env-var convention used in our projects
```
PUSHOVER_APP_TOKEN=...   # per-project app token (ev-dashboard, award-search)
PUSHOVER_USER_KEY=...
# plex-automation uses PUSHOVER_TOKEN / PUSHOVER_USER (older convention)
```
