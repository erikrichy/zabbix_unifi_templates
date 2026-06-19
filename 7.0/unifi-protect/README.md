# UniFi Protect — Zabbix Template

Zabbix 7.0 template for monitoring **UniFi Protect** (NVR + cameras) via the official local Integration API, using HTTP Agent items only — no external scripts, no Zabbix proxy, and no software running on the controller beyond what UniFi OS already provides.

> **Scope note:** This template covers the **Protect application** only — NVR identity/model, application version, and camera inventory/connectivity. It does not cover UniFi Network (switches, APs, clients — see the separate `UniFi Network API by HTTP` template), UniFi Access, storage/recording health, motion/smart-detection events, or video stream quality. It is a connectivity/inventory layer, not a full video-surveillance health template.

## What the template monitors

### Protect application / NVR
- Protect application version (with a trigger on version change, e.g. after an update)
- NVR name and model key
- API availability (trigger on `nodata`, i.e. the Protect API has stopped responding)

### Cameras
- Total number of adopted cameras (regardless of connection state)
- Number of cameras currently `CONNECTED`

### Cameras — auto-discovery
For every camera returned by the API, the following items are created automatically:
- Online state (1 = connected, 0 = disconnected/connecting)
- Camera model key (e.g. `UVC-G4-PRO`, `UVC-G5-BULLET`)

LLD macros available per camera: `{#CAM_ID}`, `{#CAM_NAME}`, `{#CAM_MAC}`, `{#CAM_MODEL}`.

## Triggers (pre-built, active by default)

| Trigger | Severity | Description |
|---|---|---|
| Protect: API unreachable (>5 min) | High | Zabbix cannot reach the Protect Integration API — check IP, API key, or controller availability |
| Protect: Application version changed | Info | Protect app was updated or downgraded |
| Camera [{#CAM_NAME}]: OFFLINE | High | A specific camera is disconnected from the controller (auto-recovers when it reconnects) |

## Template macros

| Macro | Default | Type | Description |
|---|---|---|---|
| `{$UNIFI_IP}` | `192.168.1.1` | Text | IP address of the UniFi controller (UNVR, CloudKey, Dream Machine Pro, etc.) |
| `{$UNIFI_APIKEY}` | *(empty)* | **Secret text** | API key for the Protect Integration API |
| `{$UNIFI_PORT}` | `443` | Text | HTTPS port of the controller |

`{$UNIFI_APIKEY}` is already defined with macro type **Secret text** in the template itself, so its value never shows up in the Zabbix UI or API output once set on the host. Keep it that way — don't downgrade it back to plain text when configuring the host.

## UniFi-side setup

### 1. Generate an API key

1. In the UniFi OS console, go to **Settings → Integrations** (in current UniFi OS versions the API key is generated directly under the **Integrations** section of UniFi OS, the same place used for the Network Integration API key — it is not Protect-specific, it's a controller-wide key).
2. Create a new key, name it e.g. `zabbix-protect-monitoring`.
3. Copy the value into `{$UNIFI_APIKEY}` on the Zabbix host.

This key only needs read access to query Protect's integration endpoints — there is currently no separate read-only/write distinction for these controller-wide API keys, so treat the key itself as a secret with the same care as a password, store it only as Secret text in Zabbix, and avoid reusing the same key across unrelated tools.

### 2. Verify connectivity manually (optional, recommended before importing)

```
curl -k -X GET 'https://<controller-ip>/proxy/protect/integration/v1/meta/info' \
  -H 'X-API-KEY: YOUR_KEY' \
  -H 'Accept: application/json'
```

A successful response confirms the IP, port, and API key are all correct before you wire them into Zabbix.

### 3. Network reachability

Zabbix server/proxy needs HTTPS access to the controller's `{$UNIFI_IP}:{$UNIFI_PORT}`. The template ships with `verify_peer: 'NO'` and `verify_host: 'NO'` on all HTTP Agent items, since most UniFi controllers present a self-signed certificate on their local management IP. If your controller is reachable via a hostname with a valid certificate (e.g. behind a reverse proxy), you can tighten this by switching `verify_peer`/`verify_host` to `YES` and pointing `{$UNIFI_IP}` at that hostname instead.

## Zabbix-side setup

1. Import `unifi_protect.yaml` (Data collection → Templates → Import)
2. Link the template to the host representing the UniFi Protect controller (this can be the same host as the Network template, or a separate one — they don't conflict)
3. On the host, override `{$UNIFI_IP}`, `{$UNIFI_APIKEY}`, and `{$UNIFI_PORT}` if it differs from `443`
4. Watch the first runs of `unifi.protect.meta.raw`, `unifi.protect.nvr.raw`, and `unifi.protect.cameras.raw` — errors here are almost always the API key or IP/port being wrong, or the controller being unreachable from the Zabbix server/proxy
5. Camera discovery populates automatically once `unifi.protect.cameras.raw` returns data; LLD runs with `lifetime: 7d`, so cameras removed from the controller disappear from host inventory after 7 days

## Known limitations

- The controller-wide API key used here is not scoped to Protect only — anyone holding it can query whatever Integration APIs are exposed on that controller (Network, Protect, etc., depending on what's enabled). Treat it as a sensitive credential regardless of which template uses it.
- `unifi.protect.cameras.count` and the per-camera state come from a single `cameras` array snapshot polled once a minute (`delay: 60s`); transient disconnects shorter than the polling interval won't be visible.
- No recording/storage health, motion event, or stream-quality metrics are included — only application/NVR/camera connectivity and inventory.
- Self-signed certificate verification is disabled by default (`verify_peer`/`verify_host: NO`); if you tighten this for a hostname-based setup, test connectivity again after the change.

## Suggested extensions (optional, for consideration)

- **NVR storage/disk health**: if/when the Integration API exposes storage utilization or recording retention metrics, add dependent items off a new master item rather than overloading the existing ones.
- **Per-camera discovery enrichment**: extend `lld_macro_paths` with additional fields already present in the cameras payload (e.g. firmware version, IP address) if you want inventory-style items per camera beyond online/model.
- **Doorbell-specific items**: the `nvrs` payload includes `doorbellSettings` for doorbell-capable NVRs — could be split into its own dependent items if relevant to your deployment.
- **Zabbix Actions** wired to Slack/Teams/Email for the "Camera OFFLINE" and "API unreachable" triggers specifically, since these are the most actionable for a Protect deployment.
- **Cross-reference with UniFi Access**: if cameras are tied to door events (e.g. via Access Hub), consider whether camera-offline alerts should be correlated with door-control monitoring in a future template.
- **Separate API key per controller**, if you monitor multiple Protect/Network controllers — keeps revocation scoped to one environment.

## License

MIT (or update to match your repository's license).
