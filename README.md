# Homely API - Unofficial Documentation

[![OpenAPI](https://img.shields.io/badge/OpenAPI-3.0.3-green.svg)](https://spec.openapis.org/oas/v3.0.3)
[![Docs](https://img.shields.io/badge/docs-swagger-blue.svg)](https://YOUR-USERNAME.github.io/homely-api-docs/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

> Unofficial OpenAPI specification for the [Homely](https://homely.no/) Smart Home API

[**📖 View Docs**](https://bjafl.github.io/homely-api-docs/) • [**🐛 Issues**](https://github.com/bjafl/homely-api-docs/issues) 

---

## Why This Exists

The official Homely SDK docs are incomplete (~30% coverage). This project provides:
- ✅ REST API endpoints
- ✅ WebSocket/Socket.IO event structures
- ✅ Data models with known fields

**Sources**: Official PDF from Homely + community projects

---

## Quick Start

Since the API is in beta, you'll need to [contact Homely customer service](https://www.homely.no/kundeservice/?gad_campaignid=21674806763) to get access first!

### Authentication & Basic Usage

```bash
# 1. Get access token
curl -X POST https://sdk.iotiliti.cloud/homely/oauth/token \
  -H "Content-Type: application/json" \
  -d '{"username": "{your_email}", "password": "{your_password}"}'

# 2. List locations
curl https://sdk.iotiliti.cloud/homely/locations \
  -H "Authorization: Bearer {token}"


# 3. Get devices and states
curl https://sdk.iotiliti.cloud/homely/home/{locationId} \
  -H "Authorization: Bearer {token}"
```

### Python Example

```python
import requests
import socketio

# REST API
token = requests.post('https://sdk.iotiliti.cloud/homely/oauth/token', 
    json={'username': '...', 'password': '...'}).json()['access_token']

locations = requests.get('https://sdk.iotiliti.cloud/homely/locations',
    headers={'Authorization': f'Bearer {token}'}).json()

devices = requests.get('https://sdk.iotiliti.cloud/homely/home/{locationId}',
    headers={'Authorization': f'Bearer {token}'}).json()

# WebSocket (Socket.IO - NOT raw WebSocket)
sio = socketio.Client()

@sio.on('event')
def on_event(data):
    print(data['type'], data['data'])

url = f"https://sdk.iotiliti.cloud?locationId={loc_id}&token=Bearer%20{token}"
sio.connect(url, headers={'Authorization': f'Bearer {token}', 'locationId': loc_id})
```

---

## Important Notes

- **WebSocket**: Use Socket.IO library, not raw WebSocket
- **Bearer token**: URL-encode space as `%20` between `Bearer` and _token_ in query param
- The API is read only!
- Some IOT devices connected to the Homely hub will not appear in the device list provided from the api.
  
---

## API Overview

**Base URL**: `https://sdk.iotiliti.cloud/homely`

### REST Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/oauth/token` | Get access token (expires in 60s) |
| POST | `/oauth/refresh-token` | Refresh token (valid 1800s) |
| GET | `/locations` | List user's locations |
| GET | `/home/{locationId}` | Get all devices & states |
| GET | `/alarm/state/{locationId}` | Get alarm state only ⭐ |

### WebSocket Events

**Connection**: `https://sdk.iotiliti.cloud?locationId={id}&token=Bearer%20{token}`

**Events**:
- `device-state-changed` - Sensor/device updates
- `alarm-state-changed` - Alarm system changes

---

## Alarm States

| State | Description |
|-------|-------------|
| `DISARMED` | System off |
| `ARMED_AWAY` | Full protection (away) |
| `ARMED_NIGHT` | Night mode |
| `ARMED_PARTLY` | Partial (stay home) |
| `BREACHED` | Alarm triggered |
| `*_PENDING` | Arming/exit delay |

---

## Contributing

Found issues? Discovered new endpoints? PRs welcome!

**Development**:
```bash
# Validate spec
swagger-cli validate openapi.yaml

# Preview locally
redocly preview-docs openapi.yaml
```

---

## Files

- `openapi.yaml` - Complete OpenAPI 3.0.3 spec
- `QUICK_REFERENCE.md` - API quick reference

---

## Acknowledgments

- [hansrune/homely-tools](https://github.com/hansrune/homely-tools) - Pioneering MQTT integration
- [bjafl/homely-ha](https://github.com/bjafl/homely-ha) - owner's custom home assistant integration
- [yusijs/homely-mqtt](https://github.com/yusijs/homely-mqtt) - typescript mqtt implementation with api docs


---

## Note

**Unofficial** documentation - not affiliated with Homely AS.

Use at your own risk. No warranties provided. MIT License.

