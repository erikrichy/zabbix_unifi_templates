# UniFi Zabbix Templates

Zabbix monitoring templates for UniFi controllers using the official local Integration API.
No additional software required on the Zabbix server — all data collection uses the built-in HTTP Agent.

---

## Requirements

- Zabbix 7.0+
- UniFi controller with Protect (UNVR, CloudKey Gen2, Dream Machine Pro, etc.)
- API key generated in UniFi OS

---

## Generating an API Key

1. Log in to your UniFi console
2. Go to **Settings → Control Plane → Integrations → API Keys**
3. Click **Create New Key**, give it a name (e.g. `zabbix`)
4. Copy the key — it is only shown once

---

## Templates

### `7.0/unifi_protect_template.yaml`

Monitors UniFi Protect via the official local Integration API.

**API endpoints used:**

| Endpoint | Purpose |
|---|---|
| `GET /proxy/protect/integration/v1/meta/info` | Protect application version |
| `GET /proxy/protect/integration/v1/nvrs` | NVR name and model |
| `GET /proxy/protect/integration/v1/cameras` | Camera list and connection state |

**Items collected:**

| Item | Description |
|---|---|
| Protect: Application version | Current Protect application version |
| Protect: NVR name | Name of the NVR as configured in Protect |
| Protect: NVR model | Model key of the NVR (e.g. `nvr`, `udmpro`) |
| Protect: Total cameras | Total number of adopted cameras |
| Protect: Online cameras | Number of currently connected cameras |
| Camera [name]: Online (1/0) | Per-camera connection status via LLD |
| Camera [name]: Model | Per-camera model key via LLD |

**Triggers:**

| Trigger | Priority |
|---|---|
| Protect: API unreachable (>5 min) | HIGH |
| Protect: Application version changed | INFO |
| Camera [name]: OFFLINE | HIGH |

**Low Level Discovery:**
Cameras are discovered automatically. No manual configuration needed after the template is assigned to a host.

---

## Installation

1. In Zabbix go to **Data collection → Templates → Import**
2. Select the `.yaml` file and click **Import**
3. Create a new host for your UniFi controller
4. Assign the template to the host
5. Under **Macros** on the host set:

| Macro | Value | Type |
|---|---|---|
| `{$UNIFI_IP}` | IP address of your controller | Text |
| `{$UNIFI_APIKEY}` | Your API key | Secret text |
| `{$UNIFI_PORT}` | `443` | Text |

---

## Authentication

All requests use the `X-API-KEY` header as documented in the UniFi console:

```bash
curl -k -X GET 'https://YOUR_CONSOLE_IP/proxy/protect/integration/v1/meta/info' \
  -H 'X-API-KEY: YOUR_API_KEY' \
  -H 'Accept: application/json'
```

---

## Notes

- SSL certificate verification is disabled (`verify_peer: NO`) — UniFi controllers use self-signed certificates by default
- All API data is fetched every 60 seconds via three HTTP Agent master items; all other items are dependent and do not generate additional requests
- Camera state values returned by the API: `CONNECTED`, `CONNECTING`, `DISCONNECTED`

---

## Planned Templates

| File | Status |
|---|---|
| `7.0/unifi_protect_template.yaml` | ✅ Done |
| `7.0/unifi_network_template.yaml` | 🔜 Planned |
| `7.0/unifi_access_template.yaml` | 🔜 Planned  |