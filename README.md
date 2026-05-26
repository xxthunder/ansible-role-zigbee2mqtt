# Ansible Role: zigbee2mqtt

[![CI](https://github.com/xxthunder/ansible-role-zigbee2mqtt/actions/workflows/test.yml/badge.svg)](https://github.com/xxthunder/ansible-role-zigbee2mqtt/actions/workflows/test.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Ansible](https://img.shields.io/badge/Ansible-2.11%2B-blue.svg)](https://docs.ansible.com/)
[![ansible-lint](https://img.shields.io/badge/ansible--lint-passing-brightgreen.svg)](https://ansible.readthedocs.io/projects/lint/)

Install [Zigbee2MQTT](https://www.zigbee2mqtt.io/) on a Debian host from a pinned
git tag and run it under systemd. The role:

- Installs Node.js + build prerequisites and `pnpm`, clones Z2M at
  `zigbee2mqtt_version`, and builds it (`pnpm install` + `pnpm build`).
- Runs as a dedicated `zigbee2mqtt` system user (in `dialout` for serial access);
  the source tree is root-owned and read-only to the service, state lives in a
  service-owned data dir.
- Templates `configuration.yaml` (serial, MQTT, channel, frontend, HA discovery).
- Manages a `Type=notify` systemd service.

## Requirements

- A Debian host (bookworm/trixie) with `systemd` and Node.js 20+ available via
  apt.
- A reachable **MQTT broker** (`zigbee2mqtt_mqtt_server`) with credentials.
- A Zigbee **coordinator** on a stable `/dev/serial/by-id/...` path. The service
  will not stay up without one.

## Role Variables

### Required

| Variable | Description |
|---|---|
| `zigbee2mqtt_serial_port` | Coordinator path, e.g. `/dev/serial/by-id/usb-...-if00-port0`. |
| `zigbee2mqtt_mqtt_user` | MQTT username. |
| `zigbee2mqtt_mqtt_password` | MQTT password (supply via vault; `no_log`). |

### Common defaults

| Variable | Default | Description |
|---|---|---|
| `zigbee2mqtt_version` | `2.10.1` | Z2M git tag to deploy. |
| `zigbee2mqtt_serial_adapter` | `zstack` | `zstack` (CC2652) or `ember` (EFR32). |
| `zigbee2mqtt_channel` | `20` | Zigbee channel — **set before first pairing**. |
| `zigbee2mqtt_network_key` | `GENERATE` | Network key or `GENERATE`. |
| `zigbee2mqtt_mqtt_server` | `mqtt://localhost:1883` | Broker URL — `mqtts://` for TLS. |
| `zigbee2mqtt_mqtt_ca` | `""` | Path to CA file for broker-cert validation; empty disables. |
| `zigbee2mqtt_frontend_enabled` / `_port` / `_host` | `true` / `8080` / `0.0.0.0` | Web frontend. |
| `zigbee2mqtt_frontend_ssl_cert` / `_ssl_key` | `""` / `""` | Set both to enable HTTPS; leave both empty for HTTP. |
| `zigbee2mqtt_homeassistant` | `false` | Home Assistant MQTT discovery. |
| `zigbee2mqtt_base_dir` / `_data_dir` | `/opt/zigbee2mqtt/base` / `/opt/zigbee2mqtt/data` | Source clone / runtime state. Siblings, not parent/child. |

`meta/argument_specs.yml` is the authoritative variable surface and is validated
at role start.

> **Channel:** changing `zigbee2mqtt_channel` after devices are paired forces
> re-pairing them all. Pick a channel clear of other 2.4 GHz Zigbee networks
> (e.g. Hue) and your Wi-Fi channel **before** first pairing.

> **TLS:** the role does not ship cert material. Place the CA, server cert,
> and server key on the host before running, then point the three TLS
> variables at them (`zigbee2mqtt_mqtt_ca` for broker validation,
> `zigbee2mqtt_frontend_ssl_cert`/`_key` for the HTTPS frontend). HTTPS is
> enabled by the *presence* of both frontend paths — empty both = HTTP.


## Example Playbook

Plain HTTP frontend + mqtt:// broker (default):

```yaml
- hosts: zigbee
  become: true
  vars:
    zigbee2mqtt_serial_port: "/dev/serial/by-id/usb-Acme_Zigbee_Dongle-if00-port0"
    zigbee2mqtt_channel: 20
    zigbee2mqtt_mqtt_server: "mqtt://localhost:1883"
    zigbee2mqtt_mqtt_user: zigbee2mqtt
    zigbee2mqtt_mqtt_password: "{{ vault_mqtt_zigbee2mqtt_password }}"
  roles:
    - zigbee2mqtt
```

HTTPS frontend + MQTTS broker — paths point at files the caller has already
placed on the host:

```yaml
- hosts: zigbee
  become: true
  vars:
    zigbee2mqtt_serial_port: "/dev/serial/by-id/usb-Acme_Zigbee_Dongle-if00-port0"
    zigbee2mqtt_channel: 20
    zigbee2mqtt_mqtt_server: "mqtts://localhost:8883"
    zigbee2mqtt_mqtt_user: zigbee2mqtt
    zigbee2mqtt_mqtt_password: "{{ vault_mqtt_zigbee2mqtt_password }}"
    zigbee2mqtt_mqtt_ca: /opt/zigbee2mqtt/data/tls/ca.crt
    zigbee2mqtt_frontend_port: 8443
    zigbee2mqtt_frontend_ssl_cert: /opt/zigbee2mqtt/data/tls/frontend.pem
    zigbee2mqtt_frontend_ssl_key:  /opt/zigbee2mqtt/data/tls/frontend.key
  roles:
    - zigbee2mqtt
```

## Testing

The `default` molecule scenario is **syntax-check only** (`molecule syntax`): a
meaningful converge needs a physical coordinator (the service can't stay up
without one) and a heavy `pnpm` build, so full provisioning is verified manually
against a live host. ansible-lint runs at the production profile in CI.

## License

MIT
