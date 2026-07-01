# AEGIS MDM Agent Protocol

This document defines the server contract for Android/BWC device agents. It can be exercised without hardware using `server/simulate-mdm-agents.js`.

## Security Model

Agents authenticate with a device-scoped bearer token, not a user JWT. The token is returned once during enrollment and stored on the server as a SHA-256 hash.

Production deployments should set:

```bash
export AEGIS_AGENT_ENROLLMENT_KEY='change-me'
```

When set, enrollment requests must include:

```text
X-AEGIS-Enrollment-Key: <key>
```

Agent requests authenticate with either:

```text
Authorization: Bearer <device-token>
```

or:

```text
X-AEGIS-Device-Token: <device-token>
```

## Endpoints

### Enrollment Package / QR

`POST /api/mdm/agent/enrollment-package`

Commander/admin users call this from the MDM portal. The response includes `deep_link`, `qr_content`, `qr_svg`, raw `qr_payload`, and `managed_config`. The QR SVG encodes the deep link; Android Enterprise or OEM tools should consume `managed_config`.

### Enroll Agent

`POST /api/mdm/agent/enroll`

Request:

```json
{
  "serial_number": "BWC-001",
  "imei": "990000000001",
  "model": "SC880",
  "manufacturer": "OEM",
  "os_version": "Android 14",
  "device_name": "BWC 001",
  "device_type": "bwc",
  "tenant_id": 1,
  "capabilities": {
    "device_owner": true,
    "ptt_key_probe": true,
    "wifi_policy": true
  },
  "push_provider": "polling"
}
```

Response includes the device token once:

```json
{
  "device_id": "uuid",
  "token": "device bearer token",
  "enrollment_status": "enrolled",
  "poll_interval_seconds": 5
}
```

The server defaults to a 5 second polling fallback for near-real-time command response on devices
that do not have FCM configured. Operators can tune this with `AEGIS_MDM_AGENT_POLL_SECONDS`
(accepted range: 3-60 seconds).

### Check In

`POST /api/mdm/agent/checkin`

Updates heartbeat, telemetry, location, capabilities, inventory, and push token. The response includes pending commands and marks newly pending commands as `sent`. The MDM portal receives `mdm_command_queued`, `mdm_command_sent`, and `mdm_command_result` WebSocket events so QC can see whether delay is queueing, device polling, or command execution.

### Poll Commands

`GET /api/mdm/agent/commands`

Returns pending/sent commands for the authenticated device.

### Report Command Result

`POST /api/mdm/agent/commands/:id/result`

Allowed statuses:

- `delivered`
- `executed`
- `failed`

### Location Update

`POST /api/mdm/agent/location`

Request:

```json
{ "lat": 26.224, "lng": 50.588, "battery_level": 86 }
```

### Inventory Update

`POST /api/mdm/agent/inventory`

Stores latest hardware/app inventory JSON for the device.

### Push Token Update

`POST /api/mdm/agent/push-token`

Request:

```json
{ "push_token": "fcm-token", "push_provider": "fcm" }
```

## Managed Configuration

The managed configuration object is designed for Android Enterprise, OEM staging tools, or a Device Owner bootstrapper. Required production inputs are `server_base_url`, `tenant_slug`, `enrollment_endpoint`, and an out-of-band `enrollment_key` when `AEGIS_AGENT_ENROLLMENT_KEY` is configured. See `ANDROID_MDM_PROVISIONING.md` for a full sample.

## Command Lifecycle

The existing command table supports this lifecycle:

```text
pending -> sent -> delivered -> executed
                         \-> failed
pending/sent -> expired
```

Admin users queue commands through the existing endpoint:

`POST /api/mdm/commands`

Agents retrieve and execute them through the agent endpoints above.

## Push Transport

Devices can register `push_provider=fcm` and a Firebase token through `/api/mdm/agent/push-token`. Queued commands return FCM adapter metadata and a prepared FCM HTTP v1 data message.

The FCM HTTP v1 transport is implemented in `server/push/fcm.js` (Node built-ins only): it signs a service-account JWT, exchanges it for an OAuth2 access token, and POSTs to `https://fcm.googleapis.com/v1/projects/{project_id}/messages:send`. To enable real outbound delivery, set **one** of:

```bash
export FCM_SERVICE_ACCOUNT_JSON='{...service account json...}'   # or
export FCM_SERVICE_ACCOUNT_FILE=/path/to/service-account.json
```

When configured, a queued command for an `fcm` device is sent asynchronously and the
result is broadcast over WebSocket as `mdm_push_result` (`{delivered, message_name|error}`).
The send is never reported as delivered synchronously, and the command is always persisted,
so **polling remains the guaranteed fallback** — push only wakes the device sooner. When no
service account is configured, the adapter returns an honest "not configured / will poll"
result and never fabricates delivery. The legacy `FCM_SERVER_KEY` path is deprecated and not
sent.

The adapter is unit-tested end-to-end without network/credentials via an injectable transport
(`server/test-fcm.js`, `npm run test:fcm`): JWT signing/verification, token exchange, v1
send success, honest failure on rejection, and input guards.

## No-Hardware Verification

Run the focused agent protocol suite:

```bash
cd server
node test-mdm-agent.js
```

Run a simulated Android/BWC fleet:

```bash
node simulate-mdm-agents.js --devices 25 --duration 30
```

NPM aliases:

```bash
npm run test:mdm-agent
npm run test:mdm-sim
npm run simulate:mdm
```

## Android Implementation Notes

The Android APK should store the token in encrypted storage, run a foreground service for polling/check-in, and use Device Owner APIs for privileged operations. OEM-specific PTT/BWC button support should be implemented behind vendor adapter interfaces.
