# AEGIS API Usage Guide For Stress Testing

Audience: external API/stress testing team  
Purpose: describe AEGIS API usage, authentication, WebSocket behavior, test boundaries, and reporting format for an agreed stress-test window.

## Test Environment

| Item | Value |
|---|---|
| Environment | QC |
| Browser/audio base URL | `https://20.20.0.40` |
| Raw API port | `http://20.20.0.40:7999` |
| REST base | Prefer `https://20.20.0.40/api`; raw internal port is `http://20.20.0.40:7999/api` |
| WebSocket URL | Prefer `wss://20.20.0.40/ws`; raw internal port is `ws://20.20.0.40:7999/ws` |
| Test window | Provided separately |
| Allowed peak VUs | 50 |
| Allowed request rate | Up to 50 requests/second unless a lower limit is agreed for a specific test run |
| Operational field-officer target | 20 concurrent field officers |
| Media upload scope | Allowed for approved small test photos/videos |
| PTT/WebSocket scope | Allowed, including authenticated WebSocket, PTT floor, and voice/control-plane tests |
| MDM agent API scope | Allowed for simulated device-agent enrollment, check-in, command polling, location, and inventory tests |
| Destructive API scope | Not permitted unless explicitly included in the approved test scope |
| Test data prefix | `STRESS-20260701-` |
| Test contact | Provided separately |

Use HTTPS/WSS for voice, PTT, and browser media tests. Port `7999` is the raw HTTP application port; do not use `https://20.20.0.40:7999`.

## Authentication

Login:

```http
POST /api/auth/login
Content-Type: application/json

{
  "username": "stress_commander",
  "password": "<provided separately>"
}
```

Response includes:

```json
{
  "token": "<jwt>",
  "user": {
    "id": 1,
    "username": "stress_commander",
    "role": "commander"
  }
}
```

Use the token on authenticated REST calls:

```http
Authorization: Bearer <jwt>
```

Authentication rules:

- Do not repeatedly brute-force login. Login once per virtual user, then reuse the JWT.
- If testing auth limits, coordinate that as a separate test because failed logins can trigger rate limiting.
- Do not share JWTs in tickets or screenshots.

## Core REST API Areas

The exact response shape may include additional fields. Testers should tolerate unknown fields.

### System And Session

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/auth/login` | User login |
| `GET` | `/api/auth/me` | Current user profile |
| `POST` | `/api/auth/logout` | Logout/revoke session |
| `GET` | `/api/system/config-check` | Deployment readiness/config checks |

### MCX / Communications

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/mcx/groups` | List available talkgroups |
| `POST` | `/api/mcx/calls` | Start group/private/broadcast/emergency call |
| `GET` | `/api/mcx/calls?status=active` | List active calls |
| `POST` | `/api/mcx/calls/:callId/accept` | Accept call |
| `POST` | `/api/mcx/calls/:callId/reject` | Reject call |
| `POST` | `/api/mcx/calls/:callId/end` | End call |
| `GET` | `/api/c2/audience` | C2 audience groups/users |

### MCData / Attachments

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/mcdata/messages` | Inbox/thread list |
| `POST` | `/api/mcdata/messages` | Send SDS/MCData message |
| `POST` | `/api/mcdata/attachments` | Upload base64 attachment/photo/video |
| `GET` | `/api/mcdata/attachments/:fileRef/download` | Download attachment |

### Incidents / Dispatch / Map

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/incidents` | List incidents |
| `POST` | `/api/incidents` | Create incident |
| `GET` | `/api/incidents/:id` | Incident detail |
| `POST` | `/api/sos` | Create SOS/panic incident |
| `GET` | `/api/gis/units` | Live unit positions |
| `GET` | `/api/checkpoints` | Checkpoint list |

### Evidence / Recordings

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/recordings` | List recordings/media |
| `POST` | `/api/recordings` | Register recording metadata |
| `GET` | `/api/evidence` | Unified evidence library |
| `POST` | `/api/evidence/export` | Export evidence package |

### AI / RapidScan

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/ai/media/jobs` | AI media pipeline jobs |
| `GET` | `/api/ai/events` | AI repeat/watchlist events |
| `GET` | `/api/ai/faces` | Face identities/review |
| `GET` | `/api/ai/plates` | Plate identities/review |
| `POST` | `/api/frs/scan` | FRS scan/embedding submission |
| `POST` | `/api/anpr/scan` | ANPR plate submission |

### MDM Device-Agent APIs

These use a device token after enrollment, not a user JWT. See `docs/MDM_AGENT_PROTOCOL.md`.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/mdm/agent/enroll` | Enroll device agent |
| `POST` | `/api/mdm/agent/checkin` | Heartbeat/inventory/check-in |
| `GET` | `/api/mdm/agent/commands` | Poll pending commands |
| `POST` | `/api/mdm/agent/commands/:id/result` | Report command result |
| `POST` | `/api/mdm/agent/location` | Device location update |
| `POST` | `/api/mdm/agent/inventory` | Inventory update |

## WebSocket Testing

URL:

```text
ws(s)://HOST/ws
```

Authenticate after connect:

```json
{"type":"auth","token":"<jwt>","client":"stress-test"}
```

Common messages:

```json
{"type":"location_update","lat":25.2048,"lng":55.2708}
{"type":"status_update","status":"on_duty"}
{"type":"ptt_floor_request","groupId":1}
{"type":"ptt_floor_release","groupId":1}
```

Stress-test rules:

- Start with metadata WebSocket tests before PTT/media tests.
- Do not flood `ptt_audio` unless explicitly approved.
- For PTT floor tests, always send `ptt_floor_release`.
- Use realistic connection durations, not rapid connect/disconnect loops only.
- Record server-side timestamps, call IDs, group IDs, and user IDs in the report.

## Media Upload Testing

Media APIs can consume disk, CPU, and AI processing capacity. Confirm permission before testing.

Recommended limits unless the approved test scope says otherwise:

- Photo uploads: <= 2 MB each.
- Short video/BWC uploads: <= 10 MB each.
- Max concurrent media uploads: 3 to 5.
- Use test filenames prefixed with `STRESS-`.

Do not upload real personal images, real number plates, or customer evidence.

## Destructive API Policy

The following actions are not permitted unless they are explicitly included in the approved test scope:

- Delete users/groups/devices/events/incidents.
- Archive/bulk cleanup operations.
- MDM wipe, reboot, lock, Wi-Fi disable, kiosk changes.
- Evidence purge/export at large scale.
- Customer reset command.

## Suggested Stress Test Phases

1. Smoke: login, `/api/auth/me`, groups, incidents, units.
2. Read load: incidents, GIS, groups, recordings, AI summary pages.
3. Write load: create test incidents, MCData messages, location updates.
4. WebSocket load: connect authenticated users, send presence/location updates.
5. MCX control load: create/end calls at low concurrency.
6. Media load: approved small photo/video uploads only.
7. MDM simulation: check-in/poll/result loops using device tokens.

## Reporting Requirements

The test report should include:

- Environment URL and test window.
- Tool used: k6, JMeter, Postman, Locust, custom script, etc.
- VUs, ramp-up, duration, request rate.
- Per-endpoint p50/p90/p95/p99 latency.
- HTTP status distribution.
- Error samples with timestamp and correlation data.
- WebSocket connect success rate and disconnect reasons.
- Server resource usage if available: CPU, RAM, disk, network.
- Any data created, with test prefix, so it can be cleaned up.

## Minimal Curl Examples

Login:

```bash
TOKEN=$(curl -sS -X POST "$BASE/api/auth/login" \
  -H 'Content-Type: application/json' \
  -d '{"username":"stress_commander","password":"REDACTED"}' | jq -r .token)
```

List groups:

```bash
curl -sS "$BASE/api/mcx/groups" -H "Authorization: Bearer $TOKEN"
```

Create a test incident:

```bash
curl -sS -X POST "$BASE/api/incidents" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "title":"STRESS smoke incident",
    "type":"test",
    "priority":"low",
    "description":"API stress smoke test",
    "location_lat":25.2048,
    "location_lng":55.2708
  }'
```

WebSocket auth using `wscat`:

```bash
wscat -c "$WS"
> {"type":"auth","token":"TOKEN","client":"stress-test"}
> {"type":"location_update","lat":25.2048,"lng":55.2708}
```
