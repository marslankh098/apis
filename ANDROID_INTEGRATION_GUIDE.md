# AEGIS MCS — Android Integration Guide

For an Android developer integrating a field/BWC app **and/or** the MDM device agent with
the AEGIS server. This is the contract: base URLs, auth, REST + WebSocket APIs, PTT/media,
push, MDM enrollment, permissions, and the OEM/hardware touch-points.

There are **two** integration surfaces, which can ship in one APK or two:

1. **Field app** — what the officer uses: login, PTT, SOS, location, messaging, BWC upload,
   FRS/ANPR at the edge.
2. **MDM device agent** — silent background service: enroll, check-in, execute remote
   commands (lock/wipe/locate), report inventory/location. Full contract in
   `MDM_AGENT_PROTOCOL.md`; a Kotlin skeleton already exists under `/android-agent`.

---

## 0. Base setup

| Item | Value |
|---|---|
| Server base URL | `http://<HOST>:7999` (dev) / `https://<HOST>` (pilot) |
| REST base | `<base>/api` |
| WebSocket | `<base>/ws` (`ws://` or `wss://`) |
| Content type | `application/json` (except file upload = `multipart/form-data`) |
| Min Android | API 26+ (Android 8) recommended; MDM Device Owner needs API 28+ |

**Secure context is mandatory for mic/camera.** Android's `getUserMedia`/WebRTC, and in
general a trustworthy pilot, require **HTTPS/WSS** when the app talks to a server by LAN IP
(`http://192.168.x.x` is *not* a secure context). For the pilot, the server can serve TLS
directly (`AEGIS_TLS_CERT`/`AEGIS_TLS_KEY`) — use a cert the device trusts (mkcert root CA in
lab, or a real CA in production). See `deployment/mkcert-localhost.sh` and
`OPERATOR_PILOT_RUNBOOK.md`.

---

## 1. Authentication (JWT)

### Login
`POST /api/auth/login`  ·  body `{ "username": "...", "password": "..." }`

Response:
```json
{
  "token": "<JWT>",
  "user": {
    "id": 5, "username": "alpha7", "full_name": "...", "badge_number": "...",
    "role": "officer", "department": "...", "permissions": { ... },
    "radio_id": "...", "bwc_id": "SC880-047"
  }
}
```
- The JWT is valid **24h** and carries `{ userId, role, jti }`.
- Send it on every authenticated request: `Authorization: Bearer <token>`.
- `GET /api/auth/me` returns the current profile; `POST /api/auth/logout` ends the session.

### Token storage (security requirement)
Store the JWT in **Android Keystore-backed encrypted storage** (`EncryptedSharedPreferences`
or `Keystore` + AES). Never in plain prefs/logs. On 401 (`Invalid or expired token`), silently
re-login and retry once.

> Multi-tenancy is automatic: the server derives the user's `tenant_id` from the token. The app
> never sends a tenant id on normal calls — it only sees its own tenant's data.

---

## 2. Field-app REST endpoints (the common ones)

All require `Authorization: Bearer`.

| Purpose | Method · Path | Body / notes |
|---|---|---|
| Live incidents | `GET /api/incidents` | returns `{incidents, counts}` (tenant-scoped) |
| Incident detail | `GET /api/incidents/:id` | units/timeline/evidence/recordings |
| SOS / panic | `POST /api/sos` | `{location_lat, location_lng, message}` → broadcasts to all dispatch, creates a critical incident |
| Duty check-in | `POST /api/personnel/checkin` | sets officer `on_duty`, marks shift assignment |
| My groups (talkgroups) | `GET /api/mcx/groups` | tenant-scoped MCX groups |
| Start a call | `POST /api/mcx/calls` | `{type:'group'|'private'|'broadcast'|'emergency', group_id, callee_id, priority}` |
| Messaging (MCData SDS) | `POST /api/mcdata/messages` | `{recipient_id|group_id, type:'text'|'location'|'file', body, lat, lng, file_ref}`. Authorized responses return decrypted `body` plus `encrypted:true` metadata; storage is AES-256-GCM ciphertext. |
| Inbox / thread | `GET /api/mcdata/messages`, `.../conversation/:userId`, `.../group/:groupId` | |
| BWC upload (phone evidence) | `POST /api/mcdata/attachments` | `{filename, mime_type:'video/mp4', kind:'bwc_video', duration_seconds, data_b64}` creates evidence metadata and a `file_ref`; then send a file SDS with `/api/mcdata/messages`. |
| BWC upload (register existing server file) | `POST /api/recordings` | `{type:'bwc', source, source_id, duration_seconds, format, incident_id, file_path}` |
| Seal evidence | `POST /api/bwc/recordings/:id/seal` | AES-256-GCM + SHA-256 chain of custody (commander/admin) |
| Map units | `GET /api/gis/units` | tenant-scoped live unit positions |

Field-app HTML reference implementation lives in `/field-app` (wired to these same routes).

---

## 3. Real-time WebSocket protocol (`/ws`)

Open `ws(s)://<HOST>:7999/ws`. Authenticate one of two ways:

- **Query token:** `…/ws?token=<JWT>` (server replies `{type:"auth_ok"}`), or
- **First message:** `{ "type":"auth", "token":"<JWT>" }`.

All messages are JSON `{type, …}`. Inbound/outbound types your app will use:

### PTT floor control (3GPP TS 24.380 semantics)
| Send | Meaning |
|---|---|
| `{type:"ptt_floor_request", groupId}` | request to talk |
| `{type:"ptt_floor_release", groupId}` | release the floor |
| `{type:"ptt_audio", groupId, seq, chunk}` | audio chunk (base64) into the **WebSocket stopgap**; server relays it as an AES-256-GCM encrypted envelope, see §4 |

| Receive | Meaning |
|---|---|
| `{type:"ptt_floor_granted"/"ptt_floor_denied", …}` | floor arbitration result |
| `{type:"ptt_audio", from, groupId, seq, chunk}` | another member's audio |
| `{type:"emergency_call" / "sos_alert" / "incident_created"}` | high-priority broadcasts |

### Presence / location
- `{type:"location_update", lat, lng, speed, heading}` → server stores a patrol track and
  evaluates geofences; you may receive `{type:"geofence_alert", action:"entered"|"exited", …}`.
- `{type:"status_update", status:"on_duty"|"on_scene"|…}`.

The native Field App also runs a foreground realtime service after login. It keeps a background
WebSocket alive for MCData, SOS/incident, and incoming MCVideo notifications while the UI is not
focused. This is separate from MDM command delivery; MQTT/FCM remains the target path for silent
device-management commands.

Field App MQTT is a backlog decision, not a current requirement. If added, limit it to lightweight
operational events such as alert wake-ups, unit status, or metadata notifications. Do not route
PTT audio, MCVideo media, or WebRTC signalling over MQTT.

### Rugged hardware buttons

The native Field App supports normal foreground key events, known OEM broadcasts, and an optional
accessibility key service for devices that expose PTT/camera keys globally.

Current SC580 / DSJ-HYTS5A1 mapping:

- Large PTT button: `KEY_F13` / `KeyEvent.KEYCODE_F13`.
- Camera button: `KEY_VOLUMEUP` / `KeyEvent.KEYCODE_VOLUME_UP`.

AEGIS handling:

- `KEYCODE_F13` starts hold-to-talk on key down and releases PTT on key up.
- On SC580 only, `KEYCODE_VOLUME_UP` opens Field App evidence snapshot capture.
- The Field App accessibility service must be enabled once if these keys need to work while the app
  is in the background.
- OEM apps that intercept the buttons should be disabled non-destructively, not deleted.

Detailed device log and re-enable commands: `docs/devices/SC580_HARDWARE_BUTTONS.md`.

---

## 4. Voice / video — the media plane

There are **two** paths; build against the WebRTC one for real audio.

### (a) WebSocket PTT relay — stopgap (works today)
Half-duplex: send `ptt_audio` chunks while you hold the floor; play received chunks. Simple,
already supported, fine for a first integration milestone. The relay now emits encrypted
envelopes (`payload_encrypted:true`, `payload`, `encryption.iv`, `encryption.auth_tag`).
Client-side decrypt support is still needed if native Android continues to use this stopgap.
The preferred real media path is WebRTC/SFU/SRTP below.

### (b) WebRTC media plane — real full-duplex (target)
Use Android **WebRTC** (`org.webrtc:google-webrtc` / libwebrtc). The AEGIS server is the
**signalling relay** over the same `/ws`. Message flow:

```
{type:"rtc_join", room}              → server replies {type:"rtc_peers", peers:[userId...]}
{type:"rtc_offer",  room, to, sdp}   ← relayed only to peer `to`
{type:"rtc_answer", room, to, sdp}
{type:"rtc_ice",    room, to, candidate}
{type:"rtc_leave",  room}            → peers get {type:"rtc_peer_left", peerId}
```
Also receive `{type:"rtc_peer_joined", peerId}`. Build an `RTCPeerConnection`, add the mic
track, exchange offer/answer/ICE via those messages. A working browser reference is at
`/webrtc-poc` (`webrtc-poc/index.html`) and `docs/MEDIA_PLANE_POC.md`.

- **1:1 private call** = direct peer connection (mesh).
- **Group / PTT / broadcast** = the same signalling against a server **SFU** with floor
  control gating who may send (SFU step is in progress server-side; the signalling API above
  is stable). Design your audio pipeline so the "who can transmit" decision comes from the PTT
  floor-control messages in §3.
- Needs a **secure context** (HTTPS/WSS) on real devices. STUN is configured; production needs
  a **TURN** server for NAT traversal across networks.

---

## 5. Edge AI (FRS/ANPR) — what the device sends

AEGIS runs an in-house FRS/ANPR engine. The **device/BWC extracts a face embedding** and posts
it; the server does the tenant-scoped watchlist match (keeps biometrics local — privacy/
sovereignty). Contract:

`POST /api/frs/scan`
```json
{
  "checkpoint_id": 1,
  "camera_bwc_id": "SC880-047",
  "face_embedding": [/* 512 floats, L2-normalised */],
  "face_quality_score": 0.92,
  "location_lat": 26.2, "location_lng": 50.6,
  "metadata": { "candidate_name": "optional pre-identified name" }
}
```
Response `{scan_id, match_found, match:{name, category, confidence}}`. ANPR is analogous at
`POST /api/anpr/scan` with `plate_number` + image metadata.

If your device cannot run the model (no NPU), send frames/RTSP to the server-side GPU engine
instead (server build in progress). Either way the `/api/frs/scan` contract is the same — only
the location of the inference changes.

---

## 6. Command wake-up transports

### MQTT

The server can publish MDM commands to Mosquitto. Configure the backend with
`AEGIS_MQTT_URL=mqtt://HOST:1883` (defaults to `mqtt://localhost:1883`) and register the agent with:

`POST /api/mdm/agent/push-token` `{push_provider:"mqtt"}`

Command topic:

```text
aegis/mdm/devices/{device_id}/commands
```

Result topic:

```text
aegis/mdm/devices/{device_id}/results
```

Payloads still reference the persisted REST command ID. The agent should execute the command and
publish `{command_id,status,result}` with `status` as `delivered`, `executed`, or `failed`.
The current Android MDM agent implements this subscription in `MdmMqttClient` and keeps polling as
the fallback path.

### FCM

The server also supports **FCM HTTP v1**. To receive MDM commands / alerts when the app is
backgrounded without an MQTT connection:

1. Integrate Firebase, obtain the device FCM token.
2. Register it: `POST /api/mdm/agent/push-token` `{push_token, push_provider:"fcm"}` (agent
   auth) — or include it on agent check-in.
3. The server sends a **data message** (not notification) so your `FirebaseMessagingService`
   can wake the agent and immediately `GET /api/mdm/agent/commands`.

**Polling is the guaranteed fallback** — even with no MQTT/FCM, the agent picks up commands on
its next check-in. See `MDM_AGENT_PROTOCOL.md` and `ANDROID_MDM_PROVISIONING.md`.

---

## 7. MDM device agent (the silent background service)

Full API in `MDM_AGENT_PROTOCOL.md`. Summary:

1. **Provision** via the MDM portal "Agent Package" → QR / deep link (`aegis-mdm://enroll?…`) /
   managed-config JSON. The package carries `server_base_url`, `tenant_slug`,
   `enrollment_endpoint`, and (if required) an out-of-band enrollment key.
2. **Enroll:** `POST /api/mdm/agent/enroll` (+ header `X-AEGIS-Enrollment-Key` if set) → returns
   a **device bearer token once**; store it in Keystore. All later agent calls use
   `Authorization: Bearer <device-token>` (or `X-AEGIS-Device-Token`).
3. **Run loop:** `POST /api/mdm/agent/checkin` (heartbeat + telemetry + pending commands) →
   `GET /api/mdm/agent/commands` → execute → `POST /api/mdm/agent/commands/:id/result`
   (`delivered`/`executed`/`failed`). Also `…/location` and `…/inventory`.
4. **Execute commands** behind clean adapters (the skeleton already has
   `DeviceOwnerPolicyAdapter` and `OemControlAdapter`): lock, wipe, locate, ring, app
   install, Wi-Fi/kiosk policy.

Privileged actions (Wi-Fi policy, kiosk, lock/wipe, cert install) require **Android Enterprise
Device Owner** or OEM/system-app privileges — see §9.

---

## 8. Android permissions & manifest

Declare what you actually use:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.RECORD_AUDIO"/>            <!-- PTT/WebRTC -->
<uses-permission android:name="android.permission.CAMERA"/>                  <!-- BWC/FRS -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MICROPHONE"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION"/>
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>      <!-- API 33+ -->
```
- Run a **foreground service** for PTT/location/agent so the OS doesn't kill it.
- Request runtime permissions (mic, camera, fine + background location) with rationale.
- For LAN HTTP during dev, allow cleartext to your host via a `network_security_config`
  (production = HTTPS only; pin or trust the pilot CA).

---

## 9. Hardware / OEM touch-points (BWC + rugged devices)

These are device-specific — get the **OEM SDK + key-code map** from the device vendor:

- **PTT / emergency / scan buttons:** map via `onKeyDown`/media-button receiver if the OEM
  exposes them as normal key events; otherwise call the vendor SDK behind `OemControlAdapter`.
- **Phone-as-BWC recording:** normal Android camera capture can upload MP4 evidence through
  `/api/mcdata/attachments` using `kind=bwc_video`.
- **OEM BWC camera control / recording:** OEM SDK (e.g. SC880) or ONVIF/RTSP.
- **Device Owner provisioning:** for MDM policy enforcement, provision via Android Enterprise
  (QR/zero-touch) as Device Owner, or have the OEM preload the agent as a privileged system app.

What is **not** possible from a normal (non-Device-Owner) app: Wi-Fi policy, kiosk/lock-task,
silent app install, factory reset. Plan for Device Owner or OEM privileges for those.

---

## 10. Suggested build order (milestones)

1. **Auth + REST:** login, store JWT, pull incidents/units, post SOS + check-in. (1–2 days)
2. **WebSocket presence:** connect `/ws`, send `location_update`, render live incidents/alerts.
3. **PTT (stopgap):** floor request/release + `ptt_audio` over WS — half-duplex working.
4. **MDM agent:** enroll → check-in → execute `locate`/`lock` → report result (use the
   `/android-agent` skeleton).
5. **WebRTC audio:** `RTCPeerConnection` + `rtc_*` signalling for real full-duplex (needs TLS).
6. **FCM push:** register token, wake on data message.
7. **Edge FRS / BWC + OEM buttons:** vendor SDK integration, post `face_embedding`.

## Reference material in this repo
- `MDM_AGENT_PROTOCOL.md` — exact agent API + lifecycle.
- `ANDROID_MDM_PROVISIONING.md` — QR / managed config / Device Owner / FCM.
- `MEDIA_PLANE_POC.md` + `/webrtc-poc` — WebRTC signalling reference.
- `/android-agent` — Kotlin agent skeleton (protocol client, adapters).
- `/field-app` — HTML field-app wired to the same REST/WS APIs.
