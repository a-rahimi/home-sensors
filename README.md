# Home Sensors

A Docker Compose stack for collecting, storing, and visualizing sensor data.

Currently includes:

* **rtl\_433** — SDR receiver for a ThermoPro TP211B temperature sensor, built from [a-rahimi/rtl\_433](https://github.com/a-rahimi/rtl_433)
* **Mosquitto** — MQTT broker for Qingping and other MQTT devices
* **Telegraf** — subscribes to MQTT, writes to InfluxDB
* **InfluxDB 2** — time-series database (4-year retention)
* **Grafana** — dashboards and interactive plots

Supported sensors: ThermoPro TP211B (via rtl\_433); Qingping air quality meter (e.g. CGS1/CGS2) via MQTT.

## Prerequisites

* Docker and Docker Compose
* An RTL-SDR USB dongle plugged in (for rtl\_433)
* **macOS only:** `librtlsdr` installed on the host (`brew install librtlsdr`)
* **Qingping:** The device and the host running Docker must be on the same network so the device can reach the host’s MQTT port (1883).

## Quick Start

1. **Edit credentials** (optional but recommended):

   ```bash
   nano ~/home-sensors/.env
   ```

   Change `INFLUXDB_TOKEN`, `INFLUXDB_USERNAME`, and `INFLUXDB_PASSWORD` to something secure.

2. **Set up `rtl_tcp` as a service (macOS):**

   Docker on macOS cannot access USB devices directly. Instead, `rtl_tcp`
   runs natively and exposes the SDR dongle over TCP so the container can
   reach it via `host.docker.internal:1234`.

   Create a launchd agent so `rtl_tcp` starts automatically on login and
   restarts if it crashes:

   ```bash
   mkdir -p ~/Library/LaunchAgents

   RTL_TCP_PATH="$(which rtl_tcp)"

   cat > ~/Library/LaunchAgents/com.home-sensors.rtl-tcp.plist << EOF
   ```

<?xml version="1.0" encoding="UTF-8"?>

<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
   "http://www.apple.com/DTDs/PropertyList-1.0.dtd">

<plist version="1.0">
<dict>
      <key>Label</key>
      <string>com.home-sensors.rtl-tcp</string>
      <key>ProgramArguments</key>
      <array>
         <string>${RTL_TCP_PATH}</string>
         <string>-a</string>
         <string>127.0.0.1</string>
      </array>
      <key>RunAtLoad</key>
      <true/>
      <key>KeepAlive</key>
      <true/>
      <key>StandardOutPath</key>
      <string>/tmp/rtl_tcp.log</string>
      <key>StandardErrorPath</key>
      <string>/tmp/rtl_tcp.log</string>
</dict>
</plist>
EOF

# Kill any existing rtl\_tcp process, then load the service

pkill rtl\_tcp || true
launchctl load ~/Library/LaunchAgents/com.home-sensors.rtl-tcp.plist

````

Manage the service with:

| Action | Command |
|---|---|
| Stop | `launchctl unload ~/Library/LaunchAgents/com.home-sensors.rtl-tcp.plist` |
| Start | `launchctl load ~/Library/LaunchAgents/com.home-sensors.rtl-tcp.plist` |
| Check status | `launchctl list \| grep rtl` |
| View logs | `tail -f /tmp/rtl_tcp.log` |

> **Note:** On Linux you can skip this step and use USB passthrough
> instead. See *Linux USB Passthrough* below.

3. **Start the stack:**

```bash
cd ~/home-sensors
docker-compose up -d --build
````

4. **Verify rtl\_433 is receiving data:**

   ```bash
   docker-compose logs -f rtl_433
   ```

   You should see temperature readings every ~50 seconds.

5. **Open Grafana:**

   Go to <http://localhost:3000>.
   Default login: `admin` / `admin` (you'll be prompted to change the password).

   The InfluxDB datasource is pre-configured.

6. **Create a dashboard** with a Flux query like:

   ```flux
   from(bucket: "sensors")
     |> range(start: -24h)
     |> filter(fn: (r) => r._measurement == "ThermoPro-TP211B")
     |> filter(fn: (r) => r._field == "temperature_C")
     |> map(fn: (r) => ({ r with _value: (r._value * 9.0 / 5.0) + 32.0, _field: "temperature_F" }))
   ```

## Qingping air quality meter (MQTT)

To log a Qingping air quality meter to InfluxDB and Grafana:

1. **Configure the device to use your MQTT broker**
   * Go to the [Qingping developer console](https://developer.qingping.co/privatisation/config) (register/login if needed).
   * In **Private Access Config** → **Configurations**, create a new configuration.
   * In **Self-built MQTT information**, set:
     * **Host:** your machine’s LAN IP (the host where Docker runs). Use an IP address, not a hostname—the device often does not resolve DNS names and will stay disconnected otherwise.
     * **Port:** `1883`
   * In **Private Access Config** → **Device**, add your device (pair it first in the Qingping+ or Qingping IoT app), then assign this configuration to it.
   * "Push" the configuration from the cloud UI.
   * Update the MQTT configuration on the air monitor device under "Private Cloud".

2. **Verify**
   `docker-compose logs -f telegraf` — you should see activity when the device publishes.

3. **Optional: shorter report interval**\
   By default the Qingping meter may report only every 15 minutes. To switch to every 15 seconds you send one MQTT message from your computer to the broker; the meter listens on a “down” topic and applies the setting.

   **Find your meter’s MAC address**\
   Look this up on the device's "Private Cloud" panel, under "Private MQTT Setting". There, you'll see a topic like `qingping/A1:B2:C3:D4:E5:F6/up`. The MAC is `A1:B2:C3:D4:E5:F6`.

   **Change the update interval**\
   From the repo directory, run (replace `A1:B2:C3:D4:E5:F6` with your MAC):

   ```bash
   docker-compose exec mosquitto mosquitto_pub -t 'qingping/A1:B2:C3:D4:E5:F6/down' -m '{"id":1,"need_ack":1,"type":"17","setting":{"report_interval":15,"collect_interval":15,"co2_sampling_interval":15,"pm_sampling_interval":15}}'
   ```

   This causes your computer to send this one message to the Mosquitto broker; the Qingping meter is subscribed to its `.../down` topic and receives it, then updates its report interval. You only need to run this once (until the meter is reset or reconfigured).

4. **Querying InfluxDB directly**

   You browse the content of InfluxDB interactively with in the InfluxDB UI. Open <http://server.local:8086> and sign in with the credentials from `.env`.

5. **Grafana**

   * Telegraf parses the first `sensorData` element only and uses the device timestamp, so fields are **`co2_value`**, **`pm25_value`**, **`temperature_value`**, **`humidity_value`**, etc. (no array index in names). Measurement is **`qingping`**; use the **`topic`** tag to filter if you have multiple devices.
   * Example Flux for CO₂ and PM2.5:

   ```flux
   from(bucket: "sensors")
     |> range(start: -24h)
     |> filter(fn: (r) => r._measurement == "qingping")
     |> filter(fn: (r) => r._field == "co2_value" or r._field == "pm25_value")
   ```

   The **`topic`** tag holds the MQTT topic (e.g. `qingping/582D34013D73/up`). Fields include **`co2_value`** (ppm), **`pm25_value`**, **`pm10_value`**, **`temperature_value`** (°C), **`humidity_value`** (%), **`battery_value`**, **`tvoc_value`**, and corresponding **`*_status`** fields. Filter by `r.topic == "qingping/YOUR_MAC/up"` if you have multiple devices.

6. **MQTT security settings**

   For production or exposed networks, consider enabling MQTT authentication in `mosquitto/config/mosquitto.conf` (e.g. password file and `allow_anonymous false`).

## Stopping / Restarting

```bash
cd ~/home-sensors

# Stop everything (data is preserved in Docker volumes)
docker-compose down

# Restart
docker-compose up -d
```

## Rebuilding rtl\_433

If the [a-rahimi/rtl\_433](https://github.com/a-rahimi/rtl_433) fork is updated:

```bash
cd ~/home-sensors
docker-compose build --no-cache rtl_433
docker-compose up -d rtl_433
```

## Deploying to a Remote Docker Host

You can build and run the stack on a remote machine by setting `DOCKER_HOST`. Docker will SSH into the remote host and execute everything there — no registry or image transfer needed.

```bash
export DOCKER_HOST=ssh://user@remote-host

# Build and start the full stack on the remote host
cd ~/home-sensors
docker-compose up -d --build
```

All `docker` and `docker-compose` commands in that shell session now run against the remote daemon.

## Troubleshooting

| Problem | Fix |
|---|---|
| `rtl_433` can't connect / "Connection refused" | Make sure `rtl_tcp` is running on the host (`rtl_tcp -a 127.0.0.1 &`). Check that no other process is using the SDR dongle (`pkill rtl_tcp` then restart it). |
| `rtl_433` can't find the USB dongle (Linux) | Make sure no host process is using it. Check `lsusb` for the RTL-SDR device. Try `privileged: true` in `docker-compose.yml` instead of `devices:`. |
| InfluxDB says "already set up" | This is normal on restarts. The init only runs once; data persists in the `influxdb-data` volume. |
| Grafana shows "No data" | Check `docker-compose logs rtl_433` for errors. Verify the Flux query measurement name matches the sensor model. |
| Qingping data not in Grafana | Ensure the device is configured for this broker (developer console, then factory reset). Use the broker’s **LAN IP** as Host, not a hostname (device may not resolve DNS). Check `docker-compose logs telegraf`; confirm the device publishes to `qingping/+/up` and that InfluxDB has data in measurement `qingping` (bucket `sensors`). |
| Need to wipe everything and start fresh | `docker-compose down -v` removes containers **and** volumes (all data). |
