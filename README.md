# Zabbix UniFi Templates

A collection of Zabbix 7.0 templates for monitoring **Ubiquiti UniFi** infrastructure (Network and Protect) via the official local Integration APIs and HTTP Agent items — no external scripts, no Zabbix proxy, and no additional software running on the UniFi controller.

All templates in this repository share the same design philosophy:

- **HTTP Agent only.** No custom external scripts, no Python/bash polling scripts, no Zabbix sender. Everything runs as native Zabbix item types (HTTP Agent, Dependent, Calculated) with JSONPath/JavaScript preprocessing.
- **Read-only against UniFi.** Every template assumes a dedicated, read-only API key and/or local account on the UniFi controller — these templates never write to or change UniFi configuration.
- **Low-Level Discovery (LLD) for inventory that changes.** Devices, cameras, SSIDs, and switch ports are auto-discovered, so adding/removing hardware doesn't require manual Zabbix configuration.
- **Sane trigger defaults.** Triggers that are safe to enable everywhere (device offline, API unreachable, high CPU/memory) are active by default. Triggers that depend on your specific topology (e.g. per-port link state) are shipped disabled, so importing a template doesn't flood you with irrelevant alarms.

## What's in this repository

| Template | Path | Monitors |
|---|---|---|
| **UniFi Network API by HTTP** | [`7.0/unifi-network/`](7.0/unifi-network/) | Controller/application status, devices (APs, switches, gateways), clients, WiFi SSIDs, switch ports |
| **UniFi Protect** | [`7.0/unifi-protect/`](7.0/unifi-protect/) | Protect application status, NVR identity, cameras (inventory + online state) |

Each subfolder has its own `README.md` with full details: what's monitored, all macros, required UniFi-side setup (API keys, dedicated accounts) and known limitations. Start there before importing a template.

## Repository structure

```
zabbix_unifi_templates/
├── README.md                 ← you are here
├── LICENSE
└── 7.0/                      ← templates targeting Zabbix 7.0 YAML export format
    ├── unifi-network/
    │   ├── README.md
    │   └── unifi_network.yaml
    └── unifi-protect/
        ├── README.md
        └── unifi_protect.yaml
```

The top-level `7.0/` folder reflects the Zabbix template export format version, not the UniFi OS version.

## Requirements

- Zabbix 7.0 server (the YAML export format used here is specific to 7.0; importing into older Zabbix versions will likely fail or require manual adjustment)
- Network access from the Zabbix server/proxy to the UniFi controller over HTTPS
- A UniFi controller running UniFi OS (self-hosted VM, UDM/UDR, Cloud Gateway, UNVR, etc.) with the Integration API enabled
- A dedicated, read-only API key (and, for the Network template, a dedicated read-only local account) — see each template's own README for exact steps

## Quick start

1. Pick the template you need from the table above and open its README.
2. Generate the required API key(s) / local account on your UniFi controller as described there.
3. Import the corresponding `.yaml` file in Zabbix (**Data collection → Templates → Import**).
4. Link the template to a host and override the macros (API base URL/IP, API key, site ID, credentials) at the **host** level — never edit secrets directly on the template.
5. Mark any credential macro as **Secret text** in Zabbix if it isn't already.

## Versioning and compatibility

These templates are tested against UniFi OS and Zabbix 7.0 at the time of writing. UniFi's Integration API is still evolving — endpoint shapes can change between UniFi OS releases. If an item starts returning empty/error values after a UniFi OS update, check the relevant template's README for known JSONPath/JavaScript preprocessing assumptions before opening an issue.

## License

MIT — see [LICENSE](LICENSE).
