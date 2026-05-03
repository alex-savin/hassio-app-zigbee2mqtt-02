# Zigbee2MQTT Instance 02 — Home Assistant Add-on

[![CI](https://github.com/alex-savin/hassio-app-zigbee2mqtt-02/actions/workflows/ci.yml/badge.svg)](https://github.com/alex-savin/hassio-app-zigbee2mqtt-02/actions/workflows/ci.yml)

A modified copy of the [official Zigbee2MQTT Home Assistant add-on](https://github.com/zigbee2mqtt/hassio-zigbee2mqtt) with a unique slug (`zigbee2mqtt-02`) to allow running multiple Zigbee2MQTT instances side-by-side, each with its own Zigbee coordinator.

## Why?

Home Assistant does not natively support installing the same add-on twice. If you have multiple Zigbee coordinators (e.g. different protocols, separate networks for different floors, or a dedicated coordinator for testing), you need separate add-on instances with unique slugs, ports, and data paths.

## Differences from the Stock Add-on

| Setting | Stock | Instance 02 |
|---------|-------|-------------|
| Slug | `zigbee2mqtt` | `zigbee2mqtt-02` |
| Data path | `/config/zigbee2mqtt` | `/config/zigbee2mqtt-02` |
| Socat host port | 8485 | 8487 |
| Frontend internal port | 8485 | 8092 |
| Image | `ghcr.io/zigbee2mqtt/zigbee2mqtt-{arch}` | `ghcr.io/alex-savin/zigbee2mqtt-02-amd64` |
| Architectures | `aarch64`, `amd64`, `armv7`, … | `amd64` only |

> Internal container ports (8487 socat, 8092 frontend) are different from Instance 01 (8486/8091) so the two add-ons can run side-by-side without conflicts.

## Installation

1. In Home Assistant, go to **Settings → Add-ons → Add-on Store → ⋮ → Repositories**.
2. Add this repository URL:
   ```
   https://github.com/alex-savin/hassio-app-zigbee2mqtt-02
   ```
3. Install **Zigbee2MQTT Instance 02** from the store.
4. Configure:
   - Select your Zigbee coordinator (serial port or network adapter).
   - Set a **unique MQTT base topic**: `zigbee2mqtt-02` (Settings → MQTT in the Z2M frontend).

## Upgrading to a New Zigbee2MQTT Version

Use the **Release** workflow in GitHub Actions:

1. Go to **Actions → Release → Run workflow**.
2. Enter the new version (e.g. `2.10.0-1`).
3. The workflow will:
   - Update `config.json` with the new version.
   - Prepend the version to `CHANGELOG.md`.
   - Create a git tag and GitHub release.
   - Build and push Docker images to `ghcr.io`.

Alternatively, manually update `zigbee2mqtt-02/config.json` version field and push a tag.

## CI/CD

| Workflow | Trigger | What it does |
|----------|---------|--------------|
| **CI** | Push / PR / manual | Lint JSON configs, test-build the Docker image (no push) |
| **Release** | Manual dispatch (with `version`) | Bump `config.json`, prepend CHANGELOG, tag, create GitHub release, then build & push image |
| **Release** | `release: published` | Build & push image for the published release tag (no version bump) |
| **Check Upstream Release** | Daily at 06:00 UTC / manual | Compare upstream Z2M latest tag with the current addon version and auto-dispatch **Release** with `<upstream>-1` when a new version is detected |

Images are built natively on `ubuntu-latest` (amd64) using the new reusable [`home-assistant/builder/actions/build-image`](https://github.com/home-assistant/builder) action.

## Repository Structure

```
├── .github/
│   ├── workflows/
│   │   ├── ci.yml             # Build & test pipeline
│   │   ├── release.yml        # Version bump, tag, release, build & push
│   │   └── check-upstream.yml # Daily upstream Z2M version watcher
│   └── dependabot.yml         # Keep GH Actions up to date
├── common/
│   ├── Dockerfile             # Multi-stage build (identical to upstream)
│   ├── build.yaml             # HA builder base image config
│   └── rootfs/
│       └── docker-entrypoint.sh  # Add-on entrypoint
├── zigbee2mqtt-02/
│   ├── config.json            # Add-on manifest (unique slug, ports, image)
│   ├── DOCS.md                # Documentation (shown in HA UI)
│   ├── README.md              # Add-on store description
│   ├── CHANGELOG.md           # Version history
│   ├── icon.png               # Add-on icon
│   └── logo.png               # Add-on logo
├── repository.json            # HA add-on repository metadata
├── LICENSE
└── README.md                  # This file
```

## Credits

All credit goes to the [Zigbee2MQTT](https://www.zigbee2mqtt.io/) project and its maintainers. This repository is a thin configuration wrapper around the upstream project.
