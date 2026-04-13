# Debian Reporter MQTT2HA Daemon

![Project Maintenance][maintenance-shield]

[![GitHub Activity][commits-shield]][commits]

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

[![GitHub Release][releases-shield]][releases]

A simple Linux python script to query the Debian system on which it is running for various configuration and status values which it then reports via [MQTT](https://projects.eclipse.org/projects/iot.mosquitto) to your [Home Assistant](https://www.home-assistant.io/) installation. This allows you to install and run this on each of your Debian-based systems so you can track them all via your own Home Assistant Dashboard.

This is a fork of the [RPi Reporter MQTT2HA Daemon](https://github.com/ironsheep/RPi-Reporter-MQTT2HA-Daemon) adapted to work on any Debian-based system (servers, VMs, containers, etc.) rather than being limited to Raspberry Pi hardware.

This script should be configured to be run in **daemon mode** continuously in the background as a systemd service (or optionally as a SysV init script). Instructions are provided below.

## Table of Contents

On this Page:

- [Features](#features) - key features of this reporter
- [Prerequisites](#prerequisites)
- [Installation](#installation) - install prerequisites and the daemon project
- [Configuration](#configuration) - configuring the script to talk with your MQTT broker
- [Execution](#execution) - initial run by hand, then setup to run from boot
- [Integration](#integration) - a quick look at what's reported to MQTT about this system
- [Troubleshooting](#troubleshooting) - having start up issues? Check here for common problems

Additional pages:

- [Controlling your system from Home Assistant](./RMTECTRL.md) - (Optional) Set up to allow remote control from HA
- [In practice: Advertisements to HA](./HA-ADVERT.md) - Details of the actual advertisements this Daemon publishes to MQTT
- [The Associated Lovelace RPi Monitor Card](https://github.com/ironsheep/lovelace-rpi-monitor-card) - The companion Lovelace Card for displaying monitor data
- [ChangeLog](./ChangeLog) - List of changes

## Features

- Works on any Debian-based Linux system (Debian, Ubuntu, Raspberry Pi OS, etc.)
- Supports modern predictable network interface names (enp*, ens*, wlp*) as well as classic names (eth*, wlan*)
- Detects hardware model via DMI data (x86/VMs) or device-tree (ARM boards)
- Tested with Home Assistant and Mosquitto broker
- Data is published via MQTT
- MQTT discovery messages are sent so systems are automatically registered with Home Assistant (if MQTT discovery is enabled in your HA installation)
- MQTT authentication support
- No special/root privileges are required by this mechanism (unless you activate remote commanding from HA)
- Linux daemon / systemd service, sd_notify messages generated

### Device

Each device is reported as:

| Name           | Description                                            |
| -------------- | ------------------------------------------------------ |
| `Manufacturer` | System vendor (from DMI or device-tree, e.g., Dell Inc., QEMU, Raspberry Pi) |
| `Model`        | System model (from DMI or device-tree)                 |
| `Name`         | (fqdn) myhost.home                                    |
| `software ver` | OS Name, Version (e.g., bookworm 6.1.0-18-amd64)      |

### MQTT Topics

Each device is reported as five topics:

| Name            | Device Class  | Units       | Description                                                                                                                                                                    |
| --------------- | ------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `~/monitor`     | 'timestamp'   | n/a         | Is a timestamp which shows when the system last sent information, carries a template payload conveying all monitored values (**attach the lovelace custom card to this sensor!**) |
| `~/temperature` | 'temperature' | degrees C   | Shows the latest system temperature                                                                                                                                            |
| `~/disk_used`   | none          | percent (%) | Shows the percent of root file system used                                                                                                                                     |
| `~/cpu_load`    | none          | percent (%) | Shows CPU load % over the last 5 minutes                                                                                                                                       |
| `~/mem_used`    | none          | percent (%) | Shows the percent of RAM used                                                                                                                                                  |

### Monitor Topic

The monitored topic reports the following information:

| Name                | Sub-name           | Description                                                                                                             |
| ------------------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `rpi_model`         |                    | hardware model string (from DMI or device-tree)                                                                         |
| `ifaces`            |                    | comma sep list of interface types on board [w,e,b]                                                                      |
| `temperature_c`     |                    | System temperature, in [°C] (0.1°C resolution) Note: CPU temp (used by HA sensor)                                       |
| `temp_gpu_c`        |                    | GPU temperature, in [°C] (0.1°C resolution) — if available                                                              |
| `temp_cpu_c`        |                    | CPU temperature, in [°C] (0.1°C resolution)                                                                             |
| `up_time`           |                    | duration since last booted, as [days]                                                                                   |
| `last_update`       |                    | updates last applied, as [date]                                                                                         |
| `fs_total_gb`       |                    | / (root) total space in [GBytes]                                                                                        |
| `fs_free_prcnt`     |                    | / (root) available space [%]                                                                                            |
| `fs_used_prcnt`     |                    | / (root) used space [%] (used by HA sensor)                                                                             |
| `host_name`         |                    | hostname                                                                                                                |
| `fqdn`              |                    | hostname.domain                                                                                                         |
| `ux_release`        |                    | os release name (e.g., bookworm)                                                                                        |
| `ux_version`        |                    | os version (e.g., 6.1.0-18-amd64)                                                                                      |
| `reporter`          |                    | script name, version running on system                                                                                  |
| `networking`        |                    | lists for each interface: interface name, mac address (and IP if the interface is connected)                            |
| `drives`            |                    | lists for each drive mounted: size in GB, % used, device and mount point                                                |
| `cpu`               |                    | lists the model of cpu, number of cores, etc.                                                                           |
|                     | `hardware`         | - hardware identifier                                                                                                   |
|                     | `model`            | - model description string                                                                                              |
|                     | `number_cores`     | - number of cpu cores                                                                                                   |
|                     | `bogo_mips`        | - reported performance                                                                                                  |
|                     | `serial`           | - serial number (if available)                                                                                          |
|                     | `load_1min_prcnt`  | - average % cpu load during prior minute (avg per core)                                                                 |
|                     | `load_5min_prcnt`  | - average % cpu load during prior 5 minutes (avg per core)                                                              |
|                     | `load_15min_prcnt` | - average % cpu load during prior 15 minutes (avg per core)                                                             |
| `memory`            |                    | shows the RAM configuration in MB                                                                                       |
|                     | `size_mb`          | - total memory Size in MBytes                                                                                           |
|                     | `free_mb`          | - available memory in MBytes                                                                                            |
|                     | `size_swap`        | - total swap size in MBytes                                                                                             |
|                     | `free_swap`        | - available swap size in MBytes                                                                                         |
| `mem_used_prcnt`    |                    | shows the amount of RAM currently in use (used by HA sensor)                                                            |
| `reporter`          |                    | name and version of the script reporting these values                                                                   |
| `reporter_releases` |                    | list of latest reporter formal versions                                                                                 |
| `report_interval`   |                    | interval in minutes between reports from this script                                                                    |
| `throttle`          |                    | reports the throttle status value plus interpretation (RPi only, "Not throttled" on other systems)                      |
| `timestamp`         |                    | date, time when this report was generated                                                                               |

NOTE: _cpu load averages are divided by the number of cores_

## Prerequisites

An MQTT broker is needed as the counterpart for this daemon.

MQTT is huge help in connecting different parts of your smart home and setting up of a broker is quick and easy. In many cases you've already set one up when you installed Home Assistant.

## Installation

On a modern Linux system just a few steps are needed to get the daemon working.
The following example shows the installation under Debian/Ubuntu below the `/opt` directory:

First install extra packages the script needs:

### Packages for Debian / Ubuntu

```shell
sudo apt-get install git python3 python3-pip python3-tzlocal python3-sdnotify python3-colorama python3-unidecode python3-apt python3-paho-mqtt python3-requests
```

### Packages for Arch Linux

```shell
sudo pacman -S python python-pip python-tzlocal python-notify2 python-colorama python-unidecode python-paho-mqtt python-requests inetutils
```

**NOTE**: _for users of Arch Linux the number of updates available will NOT be reported (will always show as '-1'.) This is due to Arch Linux not using the apt package manager._

### With these extra packages installed, verify access to network information

The Daemon script needs access to information about how your system connects to the network. It uses `ifconfig(8)` or `ip(8)` to look up connection names and get the IP address, etc.

Let's run `ifconfig` to ensure you have it installed:

```shell
ifconfig
```

If the command is not found, install net-tools:

```shell
sudo apt-get install net-tools
```

### Now finish with the script install

Now that the extra packages are installed let's install our script and any remaining supporting python modules.

```shell
# Get a copy of the repository
sudo git clone https://github.com/itsmariodias/Debian-Reporter-MQTT2HA-Daemon.git /opt/Debian-Reporter-MQTT2HA-Daemon

# move into your new local repository
cd /opt/Debian-Reporter-MQTT2HA-Daemon

# Make sure any script requirements are installed
sudo pip3 install -r requirements.txt
```

**WARNING:** If you choose to install these files in a location other than `/opt/Debian-Reporter-MQTT2HA-Daemon`, you will need to modify some of the control files which are used when setting up to run this script automatically. The following files:

- **debian-reporter** - Sys V init script
- **isp-debian-reporter.service** - Systemd Daemon / Service description file

... need to have any mention of `/opt/Debian-Reporter-MQTT2HA-Daemon` changed to your install location **before you can run this script as a service.**

## Configuration

To match personal needs, all operational details can be configured by modifying entries within the file [`config.ini`](config.ini.dist).
The file needs to be created first:

```shell
sudo cp /opt/Debian-Reporter-MQTT2HA-Daemon/config.{ini.dist,ini}
sudo vim /opt/Debian-Reporter-MQTT2HA-Daemon/config.ini
```

You will likely want to locate and configure the following (at a minimum) in your config.ini:

```shell
fallback_domain = {if your systems don't report their fqdn correctly}
# ...
hostname = {your-mqtt-broker}
# ...
discovery_prefix = {if you use something other than 'homeassistant'}
# ...
base_topic = {your home-assistant base topic}

# ...
username = {your mqtt username if your setup requires one}
password = {your mqtt password if your setup requires one}
```

Now that your config.ini is setup let's test!

**NOTE:** *If you wish to support remote commanding of your system then you can find additional configuration steps in [Setting up System Control from Home Assistant](./RMTECTRL.md). However, to simplify your effort, please complete the following steps to ensure all is running as desired before you attempt to set up remote control.*

## Execution

### Initial Test

A first test run is as easy as:

```shell
python3 /opt/Debian-Reporter-MQTT2HA-Daemon/ISP-Debian-mqtt-daemon.py
```

**NOTE:** _it is a good idea to execute this script by hand this way each time you modify the config.ini. By running after each modification the script can tell you through error messages if it had any problems with any values in the config.ini file, or any missing values, etc._

Using the command line argument `--config`, a directory where to read the config.ini file from can be specified, e.g.

```shell
python3 /opt/Debian-Reporter-MQTT2HA-Daemon/ISP-Debian-mqtt-daemon.py --config /opt/Debian-Reporter-MQTT2HA-Daemon
```

### Preparing to run full time

In order to have your HA system know if your system is online/offline and when it last reported-in then you must set up this script to run as a system service.

**NOTE:** Daemon mode must be enabled in the configuration file (default).

### Choose Run Style

You can choose to run this script as a `systemd service` or as a `Sys V init script`.

Let's look at how to set up each of these forms:

#### Run as Systemd Daemon / Service

Set up the script to be run as a system service as follows:

```shell
sudo ln -s /opt/Debian-Reporter-MQTT2HA-Daemon/isp-debian-reporter.service /etc/systemd/system/isp-debian-reporter.service

sudo systemctl daemon-reload

# tell system that it can start our script at system startup during boot
sudo systemctl enable isp-debian-reporter.service

# start the script running
sudo systemctl start isp-debian-reporter.service

# check to make sure all is ok with the start up
sudo systemctl status isp-debian-reporter.service
```

**NOTE:** _Please remember to run the 'systemctl enable ...' once at first install, if you want your script to start up every time your system reboots!_

#### Run as Sys V init script

In this form our wrapper script located in the /etc/init.d directory and is run according to symbolic links in the `/etc/rc.x` directories.

Set up the script to be run as a Sys V init script as follows:

```shell
sudo ln -s /opt/Debian-Reporter-MQTT2HA-Daemon/debian-reporter /etc/init.d/debian-reporter

# configure system to start this script at boot time
sudo update-rc.d debian-reporter defaults

# let's start the script now, too so we don't have to reboot
sudo /etc/init.d/debian-reporter start

# check to make sure all is ok with the start up
sudo /etc/init.d/debian-reporter status
```

### Update to latest

Like most active developers, we periodically upgrade our script. Use one of the following list of update steps based upon how you are set up.

#### Systemd commands to perform update

If you are setup in the systemd form, you can update to the latest we've published by following these steps:

```shell
# go to local repo
cd /opt/Debian-Reporter-MQTT2HA-Daemon

# stop the service
sudo systemctl stop isp-debian-reporter.service

# get the latest version
sudo git pull

# reload the systemd configuration (in case it changed)
sudo systemctl daemon-reload

# restart the service with your new version
sudo systemctl start isp-debian-reporter.service

# if you want, check status of the running script
systemctl status isp-debian-reporter.service
```

#### SysV init script commands to perform update

If you are setup in the Sys V init script form, you can update to the latest we've published by following these steps:

```shell
# go to local repo
cd /opt/Debian-Reporter-MQTT2HA-Daemon

# stop the service
sudo /etc/init.d/debian-reporter stop

# get the latest version
sudo git pull

# restart the service with your new version
sudo /etc/init.d/debian-reporter start

# if you want, check status of the running script
sudo /etc/init.d/debian-reporter status
```

## Integration

When this script is running data will be published to the (configured) MQTT broker topic "`home/nodes/{sensor_name}/...`" (e.g. `home/nodes/debian-myhost/...`).

An example:

```json
{
  "info": {
    "timestamp": "2023-03-01T15:37:06-07:00",
    "rpi_model": "QEMU Standard PC (Q35 + ICH9, 2009)",
    "ifaces": "e",
    "host_name": "myserver",
    "fqdn": "myserver.home",
    "ux_release": "bookworm",
    "ux_version": "6.1.0-18-amd64",
    "ux_updates": 3,
    "up_time": "21:08",
    "up_time_secs": 76080,
    "last_update": "2023-02-28T18:11:41-07:00",
    "fs_total_gb": 32,
    "fs_free_prcnt": 81,
    "fs_used_prcnt": 19,
    "networking": {
      "enp0s3": {
        "IP": "192.168.100.196",
        "mac": "e4:5f:01:f8:18:02",
        "rx_data": 31954,
        "tx_data": 8883
      }
    },
    "drives": {
      "root": {
        "size_gb": 32,
        "used_prcnt": 19,
        "device": "/dev/root",
        "mount_pt": "/"
      }
    },
    "memory": {
      "size_mb": 1849,
      "free_mb": 1484,
      "size_swap": 100,
      "free_swap": 100
    },
    "mem_used_prcnt": 19,
    "cpu": {
      "hardware": "",
      "model": "Intel(R) Core(TM) i7-10700K CPU @ 3.80GHz",
      "number_cores": 4,
      "bogo_mips": "7600.00",
      "serial": "",
      "load_1min_prcnt": 2.5,
      "load_5min_prcnt": 1,
      "load_15min_prcnt": 0.2
    },
    "throttle": [
      "Not throttled"
    ],
    "temperature_c": 28.7,
    "temp_gpu_c": -1.0,
    "temp_cpu_c": 28.7,
    "reporter": "ISP-Debian-mqtt-daemon v1.9.x",
    "reporter_releases": "v1.8.2,v1.7.2,v1.7.3,v1.7.4",
    "report_interval": 5
  }
}
```

**NOTE:** Where there's an IP address that interface is connected. Also, there are `tx_data` and `rx_data` values which show traffic in bytes for this reporting interval for each network interface.

This data can be subscribed to and processed by your home assistant installation. How you build your dashboard from here is up to you!

## Troubleshooting

### Issue: Some of my systems don't show up in HA

Most often fix: _install the missing package._

We occasionally have reports of users with more than one system on their network but only one shows up in Home Assistant. This is most often caused when this script generates a non-unique id for the systems. This in turn is most often caused by an inability to get network interface details. Ensure that you have net-tools package installed. If you can successfully run ifconfig(8) then you have what's needed. If not then simply run `sudo apt-get install net-tools`.

### Issue: I removed the sensor from HA now the system won't come back

Most often fix: _reboot the missing system._

When you remove a sensor from Home Assistant it tells the MQTT broker to 'forget' everything it knows about the system. Some of the information is actually `stored by the MQTT broker` so it is available while the system is offline. Our Daemon script only broadcasts this `stored` information when it is first started. As a result the system will not re-appear after delete from Home Assistant until you reboot the system in question (or, alternatively, stop then restart the script).

To reboot:

```bash
sudo shutdown -r now
```

To, instead, restart the Daemon:

```bash
sudo systemctl stop isp-debian-reporter.service
sudo systemctl start isp-debian-reporter.service
```

### General debug

The daemon script can be run by hand while enabling debug and verbose messaging:

```shell
# first stop the running daemon
sudo systemctl stop isp-debian-reporter.service

# now run the daemon with Debug and Verbose options enabled
python3 /opt/Debian-Reporter-MQTT2HA-Daemon/ISP-Debian-mqtt-daemon.py -d -v
```

This lets you inspect many of the values the script is going to use and to see the data being sent to the MQTT broker.

Then remember to restart the daemon when you are done:

```shell
# now restart the daemon
sudo systemctl start isp-debian-reporter.service
```

#### Exploring MQTT state

I find [MQTT Explorer](http://mqtt-explorer.com/) to be an excellent tool to use when trying to see what's going on the MQTT messaging any MQTT enabled device.

Alternatively I also use **MQTTBox** when I want to send messages by hand to interact via MQTT.

#### Viewing the Daemon logs

When your script is being run as a Daemon it is logging. You can view the log output since last reboot with:

```bash
journalctl -b --no-pager -u isp-debian-reporter.service
```

---

> This is a fork of the [RPi Reporter MQTT2HA Daemon](https://github.com/ironsheep/RPi-Reporter-MQTT2HA-Daemon) by [Stephen M Moraco (IronSheep)](https://github.com/ironsheep). If you find this project useful, please consider supporting the original developer:
>
> [![coffee](https://www.buymeacoffee.com/assets/img/custom_images/black_img.png)](https://www.buymeacoffee.com/ironsheep) &nbsp;&nbsp; -OR- &nbsp;&nbsp; [![Patreon](./Docs/images/patreon.png)](https://www.patreon.com/IronSheep?fan_landing=true)[Patreon.com/IronSheep](https://www.patreon.com/IronSheep?fan_landing=true)

---

## Credits

This project is a fork of the [RPi Reporter MQTT2HA Daemon](https://github.com/ironsheep/RPi-Reporter-MQTT2HA-Daemon) by [Stephen M Moraco (IronSheep)](https://github.com/ironsheep), adapted for generic Debian systems.

Thank you to Thomas Dietrich for providing a wonderful pattern for the original project. His project is [miflora-mqtt-daemon](https://github.com/ThomDietrich/miflora-mqtt-daemon).

---

## Disclaimer and Legal

> This project is a community project not for commercial use.
> The authors will not be held responsible in the event of device failure or simply errant reporting of your system status.

---

### [Copyright](copyright) | [License](LICENSE)

[commits-shield]: https://img.shields.io/github/commit-activity/y/itsmariodias/Debian-Reporter-MQTT2HA-Daemon.svg?style=for-the-badge
[commits]: https://github.com/itsmariodias/Debian-Reporter-MQTT2HA-Daemon/commits/master
[maintenance-shield]: https://img.shields.io/badge/maintainer-itsmariodias-blue.svg?style=for-the-badge
[releases-shield]: https://img.shields.io/github/release/itsmariodias/Debian-Reporter-MQTT2HA-Daemon.svg?style=for-the-badge
[releases]: https://github.com/itsmariodias/Debian-Reporter-MQTT2HA-Daemon/releases
