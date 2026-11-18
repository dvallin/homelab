# Homelab

I have an old Acer laptop in the basement, this is the stuff running on it.

## Home Assistant Configuration (GitOps)

This repository contains the Git-operated foundation for Home Assistant, following a
declarative, versioned approach. The goal is to keep house logic and dashboards in Git
while letting Home Assistant manage its internal state (integrations, database, device
registry) independently.

### Directory Structure

```
/srv/homelab/                      # Git repository
├── compose.yaml                     # Docker compose file
├── homeassistant/
│   ├── configuration.yaml          # Entry point, includes packages + dashboards
│   ├── packages/
│   │   ├── house.yaml              # House mode helpers (input_select, input_boolean)
│   │   ├── heating.yaml            # future: generic_thermostat configs
│   │   ├── ventilation.yaml        # future: ventilation automation
│   │   └── presence.yaml           # future: presence detection
│   └── dashboards/
│       ├── home.yaml               # Overview dashboard (current)
│       ├── climate.yaml            # future: temperature/humidity cards
│       └── technical.yaml          # future: debugging & sensor status
└── README.md                        # You are here
```

### /srv/homeassistant/ (Runtime Data)

```
config/                            # Docker volume mounted into HA container
├── .storage/                      # persistent state, UI settings
├── automations.yaml               # UI-managed automations
├── scripts.yaml                   # UI-managed scripts
├── scenes.yaml                    # UI-managed scenes
└── home-assistant_v2.db           # history & device registry
```

The runtime directory is **not** tracked in Git. It is created and maintained by the
Home Assistant container at `/srv/homeassistant/config:/config`.

### compose.yaml

The Docker compose file mounts configuration from this repository into the HA container
as read-only binds:

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    network_mode: host
    restart: unless-stopped
    environment:
      TZ: Europe/Berlin
    volumes:
      - /srv/homeassistant/config:/config           # persistent runtime data
      - ./homeassistant/configuration.yaml:/config/configuration.yaml:ro
      - ./homeassistant/packages:/config/packages:ro
      - ./homeassistant/dashboards:/config/dashboards:ro
      - /etc/localtime:/etc/localtime:ro
```

**Important**: Before starting the container, ensure all source files and directories
exist. Docker will create undesired directories if bind mounts are missing.

### configuration.yaml

The base configuration includes packages auto-loading and a Git-managed Lovelace
dashboard:

```yaml
default_config:

homeassistant:
  packages: !include_dir_named packages

# UI-managed configurations
automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml

# Git-managed dashboards
lovelace:
  mode: storage
  dashboards:
    haus-uebersicht:
      mode: yaml
      title: Haus
      icon: mdi:home
      show_in_sidebar: true
      filename: dashboards/home.yaml
```

- `mode: storage` keeps the default dashboard editable via the UI
- `!include_dir_named packages` automatically loads every YAML file from `packages/`
- Existing integrations (FRITZ!Box, Roborock, Brother) remain in HA's internal state

### First Package: house.yaml

Defines two core helpers that serve as the central state for later automations:

```yaml
input_select:
  house_mode:
    name: Hausmodus
    icon: mdi:home-switch
    options:
      - Zuhause
      - Abwesend
      - Urlaub
    initial: Zuhause

input_boolean:
  house_automations_enabled:
    name: Hausautomationen aktiviert
    icon: mdi:robot
    initial: false
```

- `input_select.house_mode`: Which house mode is active? (Zuhause / Abwesend / Urlaub)
- `input_boolean.house_automations_enabled`: May comfort automations run?

Both appear as entities in Home Assistant and can be controlled from the dashboard.
The `initial` values are conservative (Zuhause / false) — for production use they
can be removed so the last saved state is restored on restart.

### First Dashboard: home.yaml

An overview dashboard that works without Zigbee sensors:

```yaml
title: Haus

views:
  - title: Übersicht
    path: overview
    icon: mdi:home

    cards:
      - type: markdown
        content: |
          # Unser Haus

          Zentrale Übersicht für Raumklima,
          Heizung und Hausautomation.

      - type: entities
        title: Haussteuerung
        show_header_toggle: false
        entities:
          - entity: input_select.house_mode
            name: Hausmodus
          - entity: input_boolean.house_automations_enabled
            name: Automationen

      - type: markdown
        title: Geräte
        content: |
          {% set vacuums = states.vacuum | list %}

          **Roborock**

          {% if vacuums %}
          {% for v in vacuums %}
          - {{ v.name }}: {{ v.state }}
          {% endfor %}
          {% else %}
          Noch kein Saugroboter gefunden.
          {% endif %}

          **Weitere Geräte**

          FRITZ!Box und Brother lassen sich
          ergänzen, sobald ihre Entitäts-IDs
          feststehen.

      - type: entities
        title: System
        entities:
          - entity: sun.sun
            name: Sonnenstand
```

- Roborock section dynamically finds `vacuum` entities
- Geräte section is a placeholder until actual entity IDs are known
- System shows the sun position as a starting point

### Entity Naming Convention

For stable interfaces between declarative logic and devices, this setup uses a
lightweight naming convention. The actual devices are still bound in Home Assistant
via the UI and renamed here:

| Zweck           | Beispiel                |
|-----------------|-------------------------|
| Raumtemperatur  | `sensor.wohnzimmer_temperature` |
| Luftfeuchtigkeit| `sensor.wohnzimmer_humidity`     |
| FBH-Ausgang     | `switch.fbh_wohnzimmer`          |
| Thermostat      | `climate.fbh_wohnzimmer`         |
| Vorlauftemperatur| `sensor.fbh_vorlauf_temperature` |

This is the most important GitOps principle here: not every device needs declarative
configuration, but custom house logic should remain independent of specific hardware.

### Development Path

Start with the three files provided:

1. `configuration.yaml`
2. `packages/house.yaml`
3. `dashboards/home.yaml`

Then expand incrementally:

- Add `heating.yaml` → `generic_thermostat` for rooms
- Add `climate.yaml` → temperature history & heating states
- Add `technical.yaml` → debugging, sensor failures, relay states
- Add `presence.yaml` → reliable presence detection
- Add more dashboards (`climate.yaml`, `technical.yaml`, etc.)
- Integrate Zigbee temperature/humidity sensors
- Add actual automations (heating control, ventilation, etc.)

The house mode (`input_select.house_mode`) and automation enable switch
(`input_boolean.house_automations_enabled`) are the central gatekeepers for all
future logic.
