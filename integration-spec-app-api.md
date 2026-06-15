# Homely HA Integration — App API Spec & Implementation Plan

> Verified: June 2026 via HTTP proxy analysis of official Homely iOS app (v428).
> All endpoints confirmed working against `api.homely.no`.

---

## Background

The public Homely SDK API (`sdk.iotiliti.cloud`) has been rate-limited aggressively
since Homely's system update in June 2026, making third-party integrations unreliable.
The official Homely app uses a separate, richer private API at `api.homely.no` which
appears to have no rate limiting issues.

This spec describes how to build a Home Assistant integration using the app API.

---

## 1. Authentication

### Login

```
POST https://api.homely.no/oauth/v2/token
Content-Type: application/json

{
  "client_id": "account",
  "client_secret": "71fb00d6-ad04-43ca-96f4-fb797259da65",
  "grant_type": "password",
  "username": "<user email>",
  "password": "<user password>"
}
```

**Response:**
```json
{
  "access_token": "<JWT>",
  "refresh_token": "<JWT>",
  "token_type": "Bearer",
  "expires_in": 1800
}
```

- Access token: Keycloak JWT, `azp: account`, 30 min TTL
- Refresh token: long-lived, used to renew access without re-login

### Token Refresh

```
POST https://api.homely.no/oauth/v2/refresh-token
Content-Type: application/json

{
  "client_id": "account",
  "client_secret": "71fb00d6-ad04-43ca-96f4-fb797259da65",
  "grant_type": "refresh_token",
  "refresh_token": "<refresh_token>"
}
```

**Strategy:** Refresh proactively at 25 minutes (5 min before expiry).
Store both tokens in HA config entry data.

---

## 2. Data Model

### Locations

```
GET https://api.homely.no/locations
Authorization: Bearer <token>
```

Returns array of locations. Each has `locationId`, `name`, `role`, `companyId`.

### Home State

```
GET https://api.homely.no/home/{locationId}
Authorization: Bearer <token>
```

**Key fields (differs from SDK API):**

| Field | Type | Notes |
|---|---|---|
| `alarmState` | string | Top-level — NOT `.alarm.state` |
| `location` | object | Full hierarchy: location → floors → rooms |
| `gateway` | object | Hub details, firmware, power, Zigbee MAC |
| `devices` | array | All sensors/devices |
| `arcServiceStatus` | string | `ACTIVE` / `INACTIVE` |

**`alarmState` values:**
`DISARMED`, `ARMED_AWAY`, `ARMED_NIGHT`, `ARMED_PARTLY`, `BREACHED`,
`ALARM_PENDING`, `ALARM_STAY_PENDING`, `ARMED_NIGHT_PENDING`, `ARMED_AWAY_PENDING`

### Device Features

Each device has a `features` dict. Known feature keys:

| Feature | States | HA entity type |
|---|---|---|
| `alarm` | `alarm` (bool) | `binary_sensor` (door/motion/smoke) |
| `battery` | `defect`, `low`, `voltage` | `binary_sensor` + `sensor` |
| `temperature` | `temperature` | `sensor` |
| `diagnostic` | `online`, `tamper`, `signalStrength` | `binary_sensor` / `sensor` |
| `siren` | `alarm` | `binary_sensor` |
| `panel` | (keypad states) | (informational) |
| `armevent` | (arm/disarm events) | (informational) |

### Real-Time Updates — WebSocket

The app connects to Socket.IO at `api.homely.no` (not `sdk.iotiliti.cloud`):

```
GET wss://api.homely.no/socket.io/?EIO=3&transport=websocket
```

Event format is expected to match SDK WebSocket events:
- `device-state-changed` — sensor state update
- `alarm-state-changed` — alarm state change

Connection observed immediately after login with the app-API access token.

---

## 3. Endpoints Summary

| Endpoint | Method | Purpose |
|---|---|---|
| `/oauth/v2/token` | POST | Login |
| `/oauth/v2/refresh-token` | POST | Token refresh |
| `/users/me` | GET | Current user (verify token) |
| `/locations` | GET | List user locations |
| `/home/{locationId}` | GET | Full home state |
| `/alarm/state/{locationId}` | GET | Alarm state only |
| `/arc/membership/{locationId}/status` | GET | ARC subscription |
| `/locations/{locationId}/services` | GET | Enabled services |
| `/gateways/{gatewayId}/networks` | GET | Gateway network info |
| `/gateways/{gatewayId}/history-log` | GET | Event history |
| `/notification/device/add` | POST | Register push token |
| `/notification/device/disconnect` | POST | Deregister push token |

---

## 4. HA Integration Architecture

### Config Flow

```
User enters: email + password
→ POST /oauth/v2/token
→ GET /locations  (show location picker if multiple)
→ Store in config_entry.data:
    {username, password, location_id, access_token, refresh_token, token_expiry}
```

### DataUpdateCoordinator

```python
class HomelyAppCoordinator(DataUpdateCoordinator):
    UPDATE_INTERVAL = timedelta(minutes=5)  # REST fallback poll

    async def _async_update_data(self):
        await self._ensure_token()          # refresh if expiring
        return await self.api.get_home(self.location_id)

    async def _ensure_token(self):
        if time.time() > self.token_expiry - 300:  # 5 min margin
            await self.api.refresh_token()
```

### WebSocket (primary updates)

Connect to `wss://api.homely.no/socket.io/?EIO=3&transport=websocket` after login.
On `device-state-changed` → update coordinator data in-place → call
`async_set_updated_data()` to push to HA entities without polling.
On disconnect → exponential backoff reconnect, fall back to polling.

### Platforms

| Platform | Source | Details |
|---|---|---|
| `alarm_control_panel` | `alarmState` | DISARMED/ARMED_AWAY/ARMED_NIGHT/ARMED_PARTLY → HA states |
| `binary_sensor` | `devices[].features.alarm.states.alarm` | door, motion, smoke, tamper |
| `sensor` | `devices[].features.temperature.states.temperature` | temperature per device |
| `sensor` | `devices[].features.battery.states.voltage` | battery voltage |
| `binary_sensor` | `devices[].features.battery.states.low` | battery low |
| `sensor` | `devices[].features.diagnostic.states.signalStrength` | RSSI |
| `sensor` | `gateway.features.status.states.firmwareVersion` | gateway firmware |
| `binary_sensor` | `gateway.features.power.states.acPower` | gateway mains power |
| `sensor` | `gateway.features.power.states.batteryPercent` | gateway battery % |

---

## 5. Implementation Plan

### Phase 1 — API client (`homely_api_app.py`)

- [ ] `HomelyAppApi` class with `aiohttp.ClientSession`
- [ ] `login(username, password) → TokenPair`
- [ ] `refresh(refresh_token) → TokenPair`
- [ ] `get_locations() → list[Location]`
- [ ] `get_home(location_id) → HomeState`
- [ ] `get_alarm_state(location_id) → str`
- [ ] Token auto-refresh wrapper (decorator or middleware)
- [ ] Rate limit handling (exponential backoff on 429, though less likely)

### Phase 2 — Config flow

- [ ] Credentials form (email + password)
- [ ] Login → location picker (if multiple)
- [ ] Store tokens in `config_entry.data`
- [ ] Options flow: change location, polling interval

### Phase 3 — Coordinator

- [ ] `HomelyAppCoordinator(DataUpdateCoordinator)`
- [ ] REST polling fallback (5 min interval)
- [ ] Token refresh in `_async_update_data`
- [ ] WebSocket connection manager
  - [ ] Connect on setup
  - [ ] `device-state-changed` → `async_set_updated_data`
  - [ ] `alarm-state-changed` → `async_set_updated_data`
  - [ ] Reconnect with exponential backoff (1s → 2s → 4s → max 60s)
  - [ ] Shutdown on `async_unload_entry`

### Phase 4 — Platforms

- [ ] `alarm_control_panel.py` — map `alarmState` → HA alarm states
- [ ] `binary_sensor.py` — door/motion/smoke/tamper/battery_low sensors
- [ ] `sensor.py` — temperature, battery %, signal strength, firmware

### Phase 5 — Polish

- [ ] `diagnostics.py` — redact tokens, expose gateway/device info
- [ ] `system_health.py` — API reachable, token valid, WebSocket connected
- [ ] HACS `hacs.json` + manifest versioning

---

## 6. Key Differences from Current `bjafl/homely-ha`

| Aspect | Current (SDK API) | New (App API) |
|---|---|---|
| Base URL | `sdk.iotiliti.cloud/homely` | `api.homely.no` |
| Auth | `POST /oauth/token` (no client_secret) | `POST /oauth/v2/token` (client_id + secret) |
| Token refresh | `POST /oauth/refresh-token` | `POST /oauth/v2/refresh-token` |
| `/locations` | Returns data (when not rate-limited) | Always returns data |
| `/home` schema | `.alarm.state` | `.alarmState` (top-level) |
| Gateway data | Not included | Full firmware/power/network states |
| Room hierarchy | Not included | Full floor → room tree |
| Rate limits | Aggressive (broken June 2026) | Not observed |
| WebSocket host | `sdk.iotiliti.cloud` | `api.homely.no` |

---

## 7. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| `client_secret` rotated by Homely | Re-capture via proxy; consider making it a config option |
| App API deprecated/changed | Monitor for 4xx responses; fall back to SDK API |
| ToS violation | Integrate at user's own risk; document clearly |
| Token refresh race condition | Use asyncio Lock around token refresh |
| WebSocket auth on reconnect | Re-login or use stored refresh_token |
