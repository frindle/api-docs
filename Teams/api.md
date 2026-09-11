# Microsoft Teams — Shifts

> **Not the public Microsoft Graph.** Despite "Shifts" also existing under `graph.microsoft.com/v1.0/teams/{id}/schedule`, these projects do **not** use Graph, OAuth app-registration, or Graph scopes. They are browser extensions that capture the bearer token the live Teams web app already sends, and replay it against Teams' **internal FirstLine-Worker (FLW) Shifts service** on `flw.teams.cloud.microsoft`. Everything below is confirmed against that extension code; the Graph equivalents are noted only where useful and flagged UNCONFIRMED.

Two repos share one extension codebase:

- **MS-Shifts-To-ICS** — public parent. Scrapes your own Shifts schedule and exports to ICS / Outlook Web / iCloud (CalDAV).
- **teams-shifts-staffing-export** — private spinoff. Same export, **plus** an anonymized team-wide roster ("Export Current Staffing") that feeds `flight-staffing-forecast`. Adds `staffing.js`; otherwise the auth + Shifts-API core is identical.

Both ship a Chrome (MV3) and a Firefox (MV2) build; the two differ only in extension-API plumbing, not in the Teams calls (see `MS-Shifts-To-ICS/extension-comparison.md`).

## Auth

**No OAuth flow, no app registration, no declared Graph scopes.** The extension never authenticates itself — it borrows the user's already-authenticated Teams session:

- A `chrome.webRequest.onBeforeSendHeaders` listener scoped to `https://flw.teams.cloud.microsoft/*` captures the full request header set of any request whose URL contains `/api/` and carries an `authorization` header (`background.js:20-39`). Requires the `webRequest` permission (Firefox MV2 also needs `webRequestBlocking`) and host permissions for `*://teams.cloud.microsoft/*` and `*://flw.teams.cloud.microsoft/*` (`extension-comparison.md` manifest blocks).
- From that captured request it derives:
  - `capturedApiBase` = `{origin}/{firstPathSegment}/api` — i.e. `https://flw.teams.cloud.microsoft/{region}/api` (the first path segment is a region/routing token) (`background.js:27-29`).
  - `capturedTenantId` from a `/tenants/{id}` path segment when present (`background.js:30-31`), with fallbacks: a `tenantId`/`tid` field on team objects, a `tenantId` on open-shift objects (the most reliable source), or a JWT decode of the captured token.
- Captured headers are replayed verbatim on every subsequent `fetch` (`background.js:220-236`). If a call returns non-OK or an HTML body (login redirect), the cached headers are cleared and the extension re-opens/reloads Teams to capture fresh ones (`background.js:198-218`, `fetchShifts` retry loop).

Delegated vs application: effectively **delegated**, as the signed-in Teams user, but via a hijacked first-party token rather than a registered client. **UNCONFIRMED:** the token's actual audience/scopes (never decoded or asserted in code).

## Endpoints

All paths are relative to `capturedApiBase` (`https://flw.teams.cloud.microsoft/{region}/api`) unless noted. All are called with the captured headers.

### `GET /users/me/teams`

Lists the user's Shifts teams. Response is read defensively as `teams` / `value` / a bare array; team ids as `t.team.id` / `t.id` / `t.teamId` (`background.js:238-247`). Empty list → surfaced as "No teams found … make sure you have a schedule set up in Teams Shifts" (Shifts is not provisioned for a team until a schedule exists). Also mined for `tenantId`.

### `GET /users/me/dataindaterange`

The main schedule read. Query params (`background.js:262-270`):

| Param | Value |
|-------|-------|
| `teamIds` | comma-joined list of the user's team ids |
| `startTime` | today 00:00, ISO |
| `endTime` | next Feb-28/29 or Aug-31 boundary (`getTargetEndDate`, ~up to ~1yr out) |
| `includeShifts` | `true` |
| `includeNotes` | `true` |
| `includeOpenShifts` | `true` |
| `includeDraft` | `true` |

Response fields consumed (`background.js:272-320`): `shifts[]` (and `Shifts[]`), `openShifts[]`/`OpenShifts[]`, `timesOff[]`/`TimesOff[]`/`timeOffRequests[]`. Per shift: `startTime`/`startDateTime`/`StartDateTime`/`start`, matching end variants, `title`/`displayName`, `shiftType` (`"Absence"` → treated as all-day time-off), `theme`, `notes`. Open shifts carry an `id` (format `OPNSHFT_…`) and sometimes `tenantId`. Time-off entries expose `reason.displayName`.

**Pagination:** follow `data.nextLink` / `data['@odata.nextLink']` until null, capped at 20 pages (`background.js:285-320`). A real fix in history: the first page is mostly availability sign-up shifts, so reading only page 1 silently dropped scheduled shifts (README v1.56).

### `GET /tenants/{tenantId}/teams/{teamId}/shiftrequests?count=25`

Per-team, pending open-shift **sign-up requests** — used to tell which open slots the user has actually signed up for (`background.js:322-353`). For each returned request, `openShiftId`/`associatedOpenShiftId`/`shiftId` is matched against open-shift ids (kept only when `/^OPNSHFT_/`). Requests read as `requests` / `shiftRequests` / `value`; changelog notes the meaningful type is `requestType: "Open"` (README v1.64).

**Pagination:** `skipToken` passed as a **URL query param** (`&skipToken=…`), page-capped at 20, and — critically — the loop **stops as soon as the returned `skipToken` repeats** (`background.js:342-351`). This query-param form is the codebase's only confirmed-working `skipToken` usage.

### `POST /teams/{teamId}/bulk/getDataInDateRange`  *(staffing-export repo only)*

Team-wide roster snapshot, backing "Export Current Staffing" (`teams-shifts-exporter-chrome/background.js:385-483`, staffing-export repo only — the parent MS-Shifts-To-ICS has no `staffing.js`). JSON body:

```json
{ "startTime": "<ISO>", "endTime": "<ISO>",
  "includeShifts": true, "includeNotes": true, "includeOpenShifts": true, "includeDraft": true }
```

If that full body is rejected it retries with a bare `{startTime,endTime}` (`STAFFING_BODY_EXTRAS`, `background.js:380-383`). One team is chosen: a hard-coded `STAFFING_PREFERRED_TEAM_ID` if present, else the first team (`background.js:403`). Response `shifts[]` is filtered to `shiftType === "Working"` (drops `"Absence"`/PTO) and reduced to `{title, startTime, endTime}` — `userId` is deliberately dropped for anonymity (`staffing.js:10-14`). Output is `{teamId, generatedAt, shifts[]}` (`staffing.js:16-18`).

**Quirk — `skipToken` must be sent BOTH ways.** Confirmed broken 2026-08-13: with `skipToken` in the POST body only, the server echoed the same token and re-returned page 1 for all 20 iterations → exact 20x-duplicated data, real pages never reached (README v1.0.3, root-caused via `data/roster.json`: 9420 raw ≈ 21× the 452 unique shifts). Fix: send `skipToken` as a URL query param **in addition to** the body, and bail the moment the cursor repeats (`background.js:426-467`).

**UNCONFIRMED:** the exact request/response contract of `bulk/getDataInDateRange` (body field names, whether the server reads the body vs query `skipToken`) was never confirmed against tenant docs — the param names mirror `dataindaterange` and were reverse-engineered (see the in-code comments at `background.js:377-379`, `440-441`).

## Quirks / gotchas

- **Transport is a hijacked first-party token, not an API grant.** If the user is not signed into Teams in the same browser, nothing works; the extension force-opens/reloads `https://teams.cloud.microsoft/` to trigger the requests it sniffs.
- **HTML body = auth failure.** Every call guards `text.trimStart().startsWith('<')` — a login-redirect HTML page returns 200, so response shape, not just status, is checked (`background.js:240,291,445`).
- **Two `skipToken` conventions.** `shiftrequests` and `bulk/getDataInDateRange` paginate by `skipToken` (query param, stop-on-repeat); `dataindaterange` paginates by `nextLink`/`@odata.nextLink`. Don't mix them.
- **Timezone.** Shift instants are parsed to epoch ms and written to ICS as UTC `Z` times (not floating local), so exports are correct regardless of device/container TZ (README v1.66). All-day events (Absence/TDO, availability sign-ups) use the UTC calendar date.
- **Open shift vs availability sign-up.** Bare time-code titles (e.g. `1430 DX`, `/^\d{4}\b/`) are availability slots and are filtered out of the schedule unless the user signed up (matched via `shiftrequests`); real posted open shifts have a position prefix (`P2 1245`). Signed-up slots render as all-day `Avl HHMM` events (README v1.55, v1.65; `background.js:355-368`).
- **8-hour gap filter.** Open shifts overlapping or within 8h of an assigned shift are dropped (`isEligibleOpenShift`, README v1.17).
- **No throttling/429 handling on the Teams calls** was observed; rate-limit handling in this codebase is on the **iCloud CalDAV** upload leg (retries + inter-request delays), not the Teams API.
- **Export → ICS mapping:** `shiftType: "Absence"` and TDO/time-off → all-day events (title from `title`/`reason.displayName`, notes preserved); working shifts → timed VEVENTs; availability sign-ups → all-day `Avl HHMM`.

## Extension approach (Chrome vs Firefox)

`MS-Shifts-To-ICS/extension-comparison.md` compares the two builds. The Teams API calls are identical; the differences are all browser plumbing: MV3 service worker + `chrome.scripting` + offscreen-document downloads + object-form CSP (Chrome) vs MV2 event page + `browser.tabs.executeScript` + direct `browser.downloads` + string CSP (Firefox), bridged by a `if (typeof browser === 'undefined') var browser = chrome;` shim. Neither approach "won" — both ship; MV2 is retained for Firefox, which lacked MV3 at the time.

## Export targets (out of scope for this doc)

Beyond the Teams reads, the extension writes to: an ICS blob download, **Outlook Web** (`outlook.office.com`/`outlook.office365.com`, via `outlook_content.js` importing the ICS), and **iCloud Calendar** over CalDAV (app-specific password, `syncToiCloud`). These are export sinks, not Teams APIs, and are documented here only for completeness.

## Integration Files

| File | Purpose |
|------|---------|
| `MS-Shifts-To-ICS/teams-shifts-exporter-chrome/background.js` | Chrome MV3 service worker — auth capture, `users/me/teams`, `dataindaterange`, `shiftrequests`, ICS build, Outlook/iCloud sync |
| `MS-Shifts-To-ICS/teams-shifts-exporter-firefox/background.js` | Firefox MV2 event-page equivalent |
| `MS-Shifts-To-ICS/teams-shifts-exporter-chrome/content.js` | Navigates the Teams SPA to Shifts so the FLW requests fire (token capture) |
| `MS-Shifts-To-ICS/teams-shifts-exporter-chrome/outlook_content.js` | Outlook Web ICS import |
| `MS-Shifts-To-ICS/extension-comparison.md` | Chrome MV3 vs Firefox MV2 differences |
| `teams-shifts-staffing-export/teams-shifts-exporter-chrome/background.js` | Same core + `fetchTeamStaffing` (`bulk/getDataInDateRange` roster) |
| `teams-shifts-staffing-export/teams-shifts-exporter-chrome/staffing.js` | Pure roster helpers — `parseStaffingShifts` (`shiftType === "Working"`, drops `userId`), `buildStaffingExport` |
| `teams-shifts-staffing-export/teams-shifts-exporter-docker/` | Headless daily-sync container build |
