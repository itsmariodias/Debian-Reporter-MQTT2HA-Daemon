# Debian Reporter advertisements to Home Assistant

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

## Debian Reporter MQTT2HA Daemon

The Debian Reporter Daemon is a simple Linux python script which queries the Debian system on which it is running for various configuration and status values which it then reports via [MQTT](https://projects.eclipse.org/projects/iot.mosquitto) to your [Home Assistant](https://www.home-assistant.io/) installation.

This is a fork of the [RPi Reporter MQTT2HA Daemon](https://github.com/ironsheep/RPi-Reporter-MQTT2HA-Daemon) by [Stephen M Moraco (IronSheep)](https://github.com/ironsheep), adapted for generic Debian systems. Please consider supporting the original developer if you find this project useful.

This page describes what is being advertised to Home Assistant.

## Table of Contents

On this Page:

- [Status Endpoints](#mqtt-status-topics) - shows the sensors offered by the Daemon
- [Control Endpoints](#mqtt-command-topics) - shows the buttons offered by the Daemon (when they are configured)

Additional pages:

- [Overall Daemon Instructions](/README.md) - This project top level README
- [Controlling your system from Home Assistant](./RMTECTRL.md) - (Optional) Set up to allow remote control from HA
- [The Associated Lovelace RPi Monitor Card](https://github.com/ironsheep/lovelace-rpi-monitor-card) - The companion Custom Lovelace Card for displaying monitor data.
- [ChangeLog](./ChangeLog) - List of changes.

## Device

The Daemon reports each device as:

| Name           | Description                                            |
| -------------- | ------------------------------------------------------ |
| `Manufacturer` | Debian                                                 |
| `Model`        | System model (from DMI or device-tree)                 |
| `Name`         | (fqdn) myhost.home                                    |
| `software ver` | OS Name, Version (e.g., bookworm 6.1.0-18-amd64)      |

## Daemon config.ini settings, [MQTT] section

There are a number of settings in our `config.ini` that affect the details of the advertisements to Home Assistant. They are all found in the `[MQTT]` section of the `config.ini`. The following are used for this purpose:

| Name          | Default                 | Description                                      |
| ------------- | ----------------------- | ------------------------------------------------ |
| `hostname`    | configured hostname     | The host name of the system                      |
| `base_topic`  | {no default}            | Set this as desired for your installation        |
| `sensor_name` | default "{SENSOR_NAME}" | If you prefer to use some other form set it here |

For the purpose of this document we'll use the following to indicate where these appear in the advertisements.

- placeholders used herein: `{HOSTNAME}`, `{BASE_TOPIC}`, and `{SENSOR_NAME}`.

**NOTE**: You can have a farm of many systems all with identical **config.ini**'s. You ONLY need to set **"base\_topic"** — for the other two the default values work great.

## MQTT Status Topics

The Daemon also reports five topics for each device:

| Name            | Device Class  | Units       | Description                                                                                                                                                                      |
| --------------- | ------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `~/monitor`     | 'timestamp'   | n/a         | Is a timestamp which shows when the system last sent information, carries a template payload conveying all monitored values (**attach the lovelace custom card to this sensor!**) |
| `~/temperature` | 'temperature' | degrees C   | Shows the latest system temperature                                                                                                                                              |
| `~/disk_used`   | none          | percent (%) | Shows the percent of root file system used                                                                                                                                       |
| `~/cpu_load`    | none          | percent (%) | Shows CPU load % over the last 5 minutes                                                                                                                                         |
| `~/mem_used`    | none          | percent (%) | Shows the percent of RAM used                                                                                                                                                    |

### The Monitor endpoint

The `~/monitor` advertisement:

```json
{
  "name": "Debian Monitor {HOSTNAME}",
  "uniq_id": "Debian-e45f01Monf81801_monitor",
  "dev_cla": "timestamp",
  "stat_t": "~/monitor",
  "val_tpl": "{{ value_json.info.timestamp }}",
  "~": "{BASE_TOPIC}/sensor/{SENSOR_NAME}",
  "avty_t": "~/status",
  "pl_avail": "online",
  "pl_not_avail": "offline",
  "ic": "mdi:linux",
  "json_attr_t": "~/monitor",
  "json_attr_tpl": "{{ value_json.info | tojson }}",
  "dev": {
    "identifiers": ["Debian-e45f01Monf81801"],
    "manufacturer": "Debian",
    "name": "Debian-{HOSTNAME-FQDN}",
    "model": "QEMU Standard PC (Q35 + ICH9, 2009)",
    "sw_version": "bookworm 6.1.0-18-amd64"
  }
}
```

**NOTE**: *in this case you'll see that {HOSTNAME} is not used for the "name:" value. Instead we use the fully qualified domain name {HOSTNAME-FQDN}. For example, a system with the name "myserver" and domain ".home" has an FQDN of "myserver.home".*

### The Temperature endpoint

The `~/temperature` advertisement:

```json
{
  "name": "Debian Temp {HOSTNAME}",
  "uniq_id": "Debian-e45f01Monf81801_temperature",
  "dev_cla": "temperature",
  "unit_of_measurement": "\u00b0C",
  "stat_t": "~/monitor",
  "val_tpl": "{{ value_json.info.temperature_c }}",
  "~": "{BASE_TOPIC}/sensor/{SENSOR_NAME}",
  "avty_t": "~/status",
  "pl_avail": "online",
  "pl_not_avail": "offline",
  "ic": "mdi:thermometer",
  "dev": {
    "identifiers": ["Debian-e45f01Monf81801"]
  }
}
```

### The Disk Used endpoint

The `~/disk_used` advertisement:

```json
{
  "name": "Debian Disk Used {HOSTNAME}",
  "uniq_id": "Debian-e45f01Monf81801_disk_used",
  "unit_of_measurement": "%",
  "stat_t": "~/monitor",
  "val_tpl": "{{ value_json.info.fs_used_prcnt }}",
  "~": "{BASE_TOPIC}/sensor/{SENSOR_NAME}",
  "avty_t": "~/status",
  "pl_avail": "online",
  "pl_not_avail": "offline",
  "ic": "mdi:sd",
  "dev": {
    "identifiers": ["Debian-e45f01Monf81801"]
  }
}
```

### The CPU Load endpoint

The `~/cpu_load` advertisement:

```json
{
  "name": "Debian Cpu Use {HOSTNAME}",
  "uniq_id": "Debian-e45f01Monf81801_cpu_load",
  "unit_of_measurement": "%",
  "stat_t": "~/monitor",
  "val_tpl": "{{ value_json.info.cpu.load_5min_prcnt }}",
  "~": "{BASE_TOPIC}/sensor/{SENSOR_NAME}",
  "avty_t": "~/status",
  "pl_avail": "online",
  "pl_not_avail": "offline",
  "ic": "mdi:cpu-64-bit",
  "dev": {
    "identifiers": ["Debian-e45f01Monf81801"]
  }
}
```

### The Memory Used endpoint

The `~/mem_used` advertisement:

```json
{
  "name": "Debian Mem Used {HOSTNAME}",
  "uniq_id": "Debian-e45f01Monf81801_mem_used",
  "unit_of_measurement": "%",
  "stat_t": "~/monitor",
  "val_tpl": "{{ value_json.info.mem_used_prcnt }}",
  "~": "{BASE_TOPIC}/sensor/{SENSOR_NAME}",
  "avty_t": "~/status",
  "pl_avail": "online",
  "pl_not_avail": "offline",
  "ic": "mdi:memory",
  "dev": {
    "identifiers": ["Debian-e45f01Monf81801"]
  }
}
```

## MQTT Command Topics

Once the commanding is enabled then the Daemon also reports the commanding interface for the system. By default we've provided examples for enabling three commands (See `config.ini.dist`.) This is what the commanding interface looks like when all three are enabled:

| Name                | Device Class | Description                                                   |
| ------------------- | ------------ | ------------------------------------------------------------- |
| `~/shutdown`        | button       | Send request to this endpoint to shut the system down         |
| `~/reboot`          | button       | Send request to this endpoint to reboot the system            |
| `~/restart_service` | button       | Send request to this endpoint to restart the Daemon service   |

### The shutdown endpoint

The `~/shutdown` Command advertisement:

```json
{
  "name": "Debian Shutdown {HOSTNAME} Command",
  "uniq_id": "Debian-e45f01Monf81801_shutdown",
  "~": "{BASE_TOPIC}/command/{SENSOR_NAME}",
  "cmd_t": "~/shutdown",
  "json_attr_t": "~/shutdown/attributes",
  "avty_t": "~/status",
  "pl_avail": "online",
  "pl_not_avail": "offline",
  "ic": "mdi:power-sleep",
  "dev": {
    "identifiers": ["Debian-e45f01Monf81801"]
  }
}
```

### The Reboot endpoint

The `~/reboot` Command advertisement:

```json
{
  "name": "Debian Reboot {HOSTNAME} Command",
  "uniq_id": "Debian-e45f01Monf81801_reboot",
  "~": "{BASE_TOPIC}/command/{SENSOR_NAME}",
  "cmd_t": "~/reboot",
  "json_attr_t": "~/reboot/attributes",
  "avty_t": "~/status",
  "pl_avail": "online",
  "pl_not_avail": "offline",
  "ic": "mdi:restart",
  "dev": {
    "identifiers": ["Debian-e45f01Monf81801"]
  }
}
```

### The Restart Service endpoint

The `~/restart_service` Command advertisement:

```json
{
  "name": "Debian Restart_Service {HOSTNAME} Command",
  "uniq_id": "Debian-e45f01Monf81801_restart_service",
  "~": "{BASE_TOPIC}/command/{SENSOR_NAME}",
  "cmd_t": "~/restart_service",
  "json_attr_t": "~/restart_service/attributes",
  "avty_t": "~/status",
  "pl_avail": "online",
  "pl_not_avail": "offline",
  "ic": "mdi:cog-counterclockwise",
  "dev": {
    "identifiers": ["Debian-e45f01Monf81801"]
  }
}
```

---

> This is a fork of the [RPi Reporter MQTT2HA Daemon](https://github.com/ironsheep/RPi-Reporter-MQTT2HA-Daemon) by [Stephen M Moraco (IronSheep)](https://github.com/ironsheep). If you find this project useful, please consider supporting the original developer:
>
> [![coffee](https://www.buymeacoffee.com/assets/img/custom_images/black_img.png)](https://www.buymeacoffee.com/ironsheep) &nbsp;&nbsp; -OR- &nbsp;&nbsp; [![Patreon](./Docs/images/patreon.png)](https://www.patreon.com/IronSheep?fan_landing=true)[Patreon.com/IronSheep](https://www.patreon.com/IronSheep?fan_landing=true)

---

## Disclaimer and Legal

> This project is a community project not for commercial use.
> The authors will not be held responsible in the event of device failure or simply errant reporting of your system status.

---

### [Copyright](copyright) | [License](LICENSE)
