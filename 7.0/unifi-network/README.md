# UniFi Network API by HTTP — Zabbix Template

Zabbix 7.0 template for monitoring **UniFi Network** infrastructure (controllers such as UDM/UDR/UNVR/Cloud Gateway, or self-hosted UniFi OS) via HTTP Agent items, without requiring external scripts, a Zabbix proxy, or agents on the UniFi devices themselves. All communication goes directly from the Zabbix server/proxy to the UniFi controller over HTTPS.

> **Scope note:** This template is primarily designed for **UniFi OS running as a VM / Network application** (i.e. the controller software itself), and focuses on **device, client, WiFi and switch-port level monitoring**. It does **not** include WAN monitoring, ISP/uplink health, firewall, gateway routing, VPN, or any other security-appliance-level checks — those belong to a separate template/scope and are intentionally out of bounds here.

The template combines **two data sources**:

1. **Official UniFi Network Integration API** (`/proxy/network/integration/v1/...`, authenticated via `X-API-KEY`) — the modern, stable REST API.
2. **Classic/internal UniFi API** (`/api/auth/login` + `/proxy/network/api/s/<site>/...`, authenticated via cookie session) — used only for data the official API does not (yet) expose, such as per-port switch detail and per-SSID client breakdown.

This hybrid approach is necessary because the official Integration API, in its current UniFi OS version, does not export individual switch port state or client-to-SSID breakdown.

## What the template monitors

### Application / controller
- UniFi Network application version
- API availability (trigger on `nodata`, i.e. data has stopped arriving)
- Number of pending (not-yet-adopted) devices

### Devices (APs, switches, gateways) — auto-discovery
For every device found on the network, the following items are created automatically:
- Online/offline state
- IP address, firmware version (inventory)
- CPU and memory utilization (%)
- Uptime
- Uplink RX/TX rate

Plus network-wide summary items:
- Number of APs online / offline
- Number of switches online / offline

### Clients
- Total client count (official API)
- WiFi client count (from classic API)
- Wired client count (derived: total − wifi)

### WiFi (SSID) — auto-discovery
For every broadcast SSID:
- Enabled/disabled state
- Broadcasting frequency bands (2.4/5/6 GHz)
- Number of connected clients per SSID

### Switch ports — auto-discovery
For every port on every device (sourced from classic API, `port_table`):
- Link state (up/down)
- Negotiated speed (bps)
- RX/TX rate (Bps)
- Error and discard counters
- PoE power draw (W)

> Port-level item prototypes and triggers are included in the template, but the **port trigger prototypes are disabled by default** — not every port in a network is worth alerting on (e.g. unused ports). After importing the template, manually enable triggers only for the ports you actually want to monitor (uplinks, ports to critical devices).

## Triggers (pre-built, active by default)

| Trigger | Severity | Description |
|---|---|---|
| UniFi API info has not been updated | High | Lost communication with the Integration API |
| UniFi devices data has not been updated | High | Lost communication — device list |
| UniFi classic client fetch has not been updated | High | Lost login/data from classic API (clients) |
| UniFi classic device fetch has not been updated | High | Lost login/data from classic API (ports) |
| UniFi device is offline: {#DEVICE_NAME} | High | A specific device is offline |
| High CPU on UniFi device | Warning | Average CPU over 10 min above threshold |
| High memory on UniFi device | Warning | Average memory over 10 min above threshold |
| UniFi has pending devices awaiting adoption | Info | A device is waiting for adoption |
| WiFi broadcast is disabled: {#SSID_NAME} | Warning | An SSID was disabled |
| Port down (port prototype) | High | **Disabled by default** |
| Port speed changed (port prototype) | Warning | **Disabled by default** |

## Template macros

| Macro | Default | Description |
|---|---|---|
| `{$UNIFI.API.BASE}` | — | Controller base URL, e.g. `https://10.4.57.1:443` or a public domain behind a reverse proxy. **No trailing slash.** |
| `{$UNIFI.API.KEY}` | `CHANGEME` | API key for the Integration API (generated in UniFi OS) |
| `{$UNIFI.SITE.ID}` | — | Site UUID for the Integration API (not "default" — the actual UUID, obtained via `/integration/v1/sites`) |
| `{$UNIFI.CLASSIC.USERNAME}` | `zabbix_readonly` | Local account used for classic API login |
| `{$UNIFI.CLASSIC.PASSWORD}` | `CHANGEME` | Password for the local account |
| `{$UNIFI.CLASSIC.SITE}` | `default` | Site name (not UUID) for classic API endpoints |
| `{$UNIFI.CLASSIC.AUTH.COOKIE}` | `TOKEN` | Name of the session cookie returned by classic login — UniFi OS typically uses `TOKEN`; older Network appliances may use `unifises` |
| `{$UNIFI.API.TIMEOUT}` | `10s` | HTTP request timeout |
| `{$UNIFI.POLL.INFO}` | `5m` | Polling interval for application info |
| `{$UNIFI.POLL.SITES}` | `30m` | Polling interval for sites |
| `{$UNIFI.POLL.DEVICES}` | `2m` | Polling interval for devices and ports |
| `{$UNIFI.POLL.CLIENTS}` | `2m` | Polling interval for clients |
| `{$UNIFI.POLL.WIFI}` | `10m` | Polling interval for WiFi broadcasts |
| `{$UNIFI.POLL.DEVICE.STATS}` | `2m` | Polling interval for per-device CPU/RAM/uptime |
| `{$UNIFI.DATA.STALE}` | `10m` | How long without data before the "nodata" trigger fires |
| `{$UNIFI.CPU.HIGH}` | `90` | Threshold for the high-CPU trigger (%) |
| `{$UNIFI.MEM.HIGH}` | `90` | Threshold for the high-memory trigger (%) |

After importing the template, override `{$UNIFI.API.KEY}`, `{$UNIFI.CLASSIC.PASSWORD}`, and ideally `{$UNIFI.CLASSIC.USERNAME}` at the **host** level (not the template), and mark them as **Secret text** macros in Zabbix so their values aren't exposed in the UI or API output to regular users.

## UniFi-side setup

### 1. Create a dedicated local account

Create a **separate local account** for this monitoring that is not used for anything else:

1. UniFi OS → **Settings → Admins & Users → Add Admin/User → Add a New User**
2. Choose a **local account** (not a UI Account/cloud SSO) — classic login (`/api/auth/login`) only works for accounts managed locally by the controller, not Ubiquiti cloud identities.
3. Assign a **Limited Admin** role (or equivalent) for the **Network** application only, with **View Only / Read-Only** permissions. The account must not have write/admin rights — it only needs to read state, never change configuration.
4. Do **not** grant the account access to other applications (Protect, Access, Talk, etc.) or to any sites other than the ones you intend to monitor.
5. Disable 2FA enforcement for this account if it would block automated login (or use whatever 2FA-exempt mechanism your UniFi OS version supports) — in practice it's simplest to leave 2FA off for this read-only service account, since it has no write access at all.
6. Use a strong, unique password and store it only in the Zabbix macro as Secret text.

> **Why a dedicated account?** Traceability (who/what is logging in), the ability to revoke access at any time without affecting human admins, and minimizing blast radius if the password ever leaks — since it's read-only, an attacker couldn't use it to change network configuration.

### 2. Generate an API key for the Integration API

1. In the UniFi OS console, go to **Settings → Integrations** (in current UniFi OS versions the API key is generated directly under the **Integrations** section of UniFi OS itself, not under Network app settings).
2. Create a new API key and name it, e.g., `zabbix-monitoring`.
3. Copy the value into `{$UNIFI.API.KEY}`.

The API key works independently of the local account above — the Integration API is a separate authentication mechanism (the `X-API-KEY` header), while the classic API needs a username/password login.

### 3. Find the Site ID

Once the `unifi.sites.raw` item runs, it returns the list of sites including their `id` (UUID). Use this value for `{$UNIFI.SITE.ID}`. Alternatively, retrieve it manually:

```
GET {base_url}/proxy/network/integration/v1/sites
Header: X-API-KEY: <your_key>
```

### 4. Network reachability

The Zabbix server/proxy needs HTTPS access to the controller's port (typically 443, or 11443 for some UniFi OS console setups). If the controller sits behind a reverse proxy with its own certificate (e.g. Let's Encrypt), check whether your Zabbix HTTP Agent items should have SSL verification enabled or disabled, depending on whether you're using a valid certificate or the controller's self-signed cert.

## Zabbix-side setup

1. Import `unifi_zabbix_template.yaml` (Data collection → Templates → Import)
2. Link the template to the host representing the UniFi controller
3. On the host, override the macros: `{$UNIFI.API.BASE}`, `{$UNIFI.API.KEY}`, `{$UNIFI.SITE.ID}`, `{$UNIFI.CLASSIC.USERNAME}`, `{$UNIFI.CLASSIC.PASSWORD}`
4. Verify `{$UNIFI.CLASSIC.AUTH.COOKIE}` — for most UniFi OS controllers the correct value is `TOKEN`; for older/other versions, check the cookie name in the `Set-Cookie` header of the `/api/auth/login` response
5. Watch the first runs of `unifi.info.raw`, `unifi.devices.raw`, `unifi.classic.clients.raw`, and `unifi.classic.devices.raw` — errors here are almost always caused by the API key, credentials, or an incorrect cookie name
6. Discovery rules (devices, SSID, ports) populate automatically after the master items run successfully for the first time; LLD runs with `lifetime: 30d`, so devices/ports/SSIDs that disappear from the network are removed from host inventory after 30 days

## Known limitations

- The classic API login (`unifi.classic.clients.raw`, `unifi.classic.devices.raw`) is implemented as a single HTTP Agent item that performs its own login and follow-up GET request inside the preprocessing steps (REGEX → JavaScript). A UniFi OS version change may alter the response shape or cookie name — if that happens, check the JavaScript preprocessing code.
- AP/switch detection (`isAP`/`isSW` functions) is heuristic (based on `model` prefixes and `features`), not 100% reliable for all future device models — for new models, review and adjust the regex/prefixes in the JavaScript preprocessing.
- Port-level trigger prototypes are intentionally disabled so that importing the template doesn't flood the environment with alarms on unused ports.
- The cookie-based session from the classic API is re-acquired on every polling cycle (no session caching) — at very high polling frequency (below 1 min) this adds load on the controller; it's not recommended to go below `{$UNIFI.POLL.CLIENTS}` = 1m.
- As noted above, this template does not cover WAN/ISP uplink health, gateway/firewall, routing, or VPN monitoring — only the Network application's device/client/WiFi/port layer.

## License

MIT (or update to match your repository's license).
