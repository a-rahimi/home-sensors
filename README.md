# Home Sensors

A Docker Compose stack for collecting, storing, and visualizing sensor data.

Currently includes:

* **rtl\_433** — SDR receiver for a ThermoPro TP211B temperature sensor, built from [a-rahimi/rtl\_433](https://github.com/a-rahimi/rtl_433)
* **InfluxDB 2** — time-series database (4-year retention)
* **Grafana** — dashboards and interactive plots

## Prerequisites

* Docker and Docker Compose
* An RTL-SDR USB dongle plugged in
* **macOS only:** `librtlsdr` installed on the host (`brew install librtlsdr`)

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
| Need to wipe everything and start fresh | `docker-compose down -v` removes containers **and** volumes (all data). |
