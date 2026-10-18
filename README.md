# ESP32-S3-4848S040 Weather, Radar & Camera Panel

A touchscreen weather, radar, and optional security-camera viewer for the **ESP32-S3-4848S040 480×480 display**. Built with [ESPHome](https://esphome.io/) and [LVGL](https://lvgl.io/), it displays current weather, animated precipitation radar, National Weather Service forecasts and alerts, air quality, and JPEG frames from up to five Frigate cameras.

**No Home Assistant dependency.** Weather and radar data are fetched directly over Wi-Fi without a local server. **The camera page requires an existing, running Frigate server** reachable on the local network. MQTT is optional for the panel and is not how camera images are delivered.

## Screenshots

These are **480×480 simulator renders**, not photographs of the physical panel. The simulator uses fixture weather values; its map/radar assets and location may differ from the device configuration. **The camera page is not represented in these screenshots**; the simulator currently renders only the four weather pages.

| Radar | Forecast |
|:---:|:---:|
| ![Radar screen](docs/screenshots/radar.png) | ![NWS forecast screen](docs/screenshots/forecast.png) |
| Conditions | Alerts |
| ![Conditions screen](docs/screenshots/conditions.png) | ![Alerts screen](docs/screenshots/alerts.png) |

## Features

- **Radar:** Dark RainViewer basemap with four recent historical radar frames, animated on a 2.4-second loop. A marker identifies the configured location. Radar is historical imagery, **not a future forecast**.
- **Current conditions:** Temperature, feels-like temperature, wind and gusts, humidity, dew point, pressure, cloud cover, precipitation, and UV index.
- **Air quality:** US AQI, PM2.5, and ozone from Open-Meteo's air-quality API.
- **NWS forecast:** Up to six forecast periods, including temperature, short description, and precipitation probability when available.
- **NWS alerts:** Active alert count and details, with a radar-page alert button when alerts are present.
- **Camera viewer:** Fifth page with Previous/Next selection among five named Frigate cameras, using periodically refreshed JPEG images rather than H.264 or real-time video playback.
- **Touch navigation:** Five pages on the physical panel — Radar, Forecast, Conditions, Alerts, Cameras.
- **Clock and power management:** SNTP clock in the configured timezone; backlight dims after one minute without LVGL activity and turns off after ten minutes. Touch wakes the backlight.
- **Fault-aware radar loading:** Frame downloads are serialized. The LVGL animation is assigned its online image sources only after all four downloads succeed, avoiding attempts to animate uninitialized image descriptors.

## Hardware

Designed for the **ESP32-S3-4848S040** board with these components:

| Component | Configuration |
|---|---|
| MCU | ESP32-S3 |
| Display | 480×480 RGB, ST7701S controller |
| Touch | GT911 over I²C |
| Flash | 16 MB |
| PSRAM | Octal, 80 MHz |
| Firmware | ESP-IDF through ESPHome |
| Display backlight | LEDC PWM on GPIO38 |

The display/touch pins in [`panel.yaml`](panel.yaml) are board-specific. Key assignments:

| Function | GPIO |
|---|---|
| Touch SDA / SCL | 19 / 45 |
| LCD SPI clock / MOSI / CS | 48 / 47 / 39 |
| LCD DE / HSYNC / VSYNC / PCLK | 18 / 16 / 17 / 21 |
| RGB red | 11, 12, 13, 14, 0 |
| RGB green | 8, 20, 3, 46, 9, 10 |
| RGB blue | 4, 5, 6, 7, 15 |

Do not assume these pins apply to other ESP32-S3 480×480 boards.

## Getting started

### 1. Install ESPHome

Install the [ESPHome command-line tools](https://esphome.io/guides/installing_esphome/) using your preferred method. For example, with `pipx`:

```bash
pipx install esphome
```

The physical configuration uses the older ESPHome `image:` / `online_image:` schema. If your installed ESPHome version rejects those options, use a compatible release or migrate the image configuration before flashing. The separate screenshot workflow pins ESPHome **2026.8.2 for its host simulator**; that does not independently verify the physical firmware configuration.

### 2. Clone and configure

```bash
git clone https://github.com/voidnullvalue/ESP32-S3-4848S040-480x480.git
cd ESP32-S3-4848S040-480x480
```

Edit [`panel.yaml`](panel.yaml):

- Set `wifi.ssid`, `wifi.password`, and the fallback access-point password. The checked-in strings are placeholders.
- Set `latitude` and `longitude` under `substitutions` for your location.
- Set `nws_latitude` and `nws_longitude` to the same coordinates rounded to four decimal places for the NWS endpoints.
- Update `time.timezone` and `http_request.useragent` as appropriate.
- Copy `secrets.example.yaml` to `secrets.yaml` and set `frigate_host` to **the Frigate server's IP or hostname**, not an individual camera's address. The firmware constructs `http://<frigate_host>:5000`.
- Set `mqtt_broker` in `secrets.yaml` if keeping the optional MQTT block. You can remove the `mqtt:` block if MQTT is not wanted.

For example:

```yaml
substitutions:
  latitude: "40.000000"    # example only: replace with your latitude
  longitude: "-105.000000" # example only: replace with your longitude
  nws_latitude: "40.0000"
  nws_longitude: "-105.0000"
  location_name: "YOUR LOCATION"
  radar_zoom: "7"
  basemap_zoom: "7"
  radar_color: "2"
```

**Keep `radar_zoom` and `basemap_zoom` equal.** The map and radar use the same Web Mercator zoom semantics; differing zoom values misalign the precipitation overlay.

From the repository root, configure the two host settings locally:

```bash
cp secrets.example.yaml secrets.yaml
${EDITOR:-vi} secrets.yaml
```

```yaml
# secrets.yaml — illustrative values; substitute your own host addresses
frigate_host: "FRIGATE_SERVER_IP_OR_HOSTNAME"
mqtt_broker: "MQTT_SERVER_IP_OR_HOSTNAME"
```

`panel.yaml` loads those values with `!secret`. The repository's `.gitignore` excludes `secrets.yaml` (and `secrets.*.yaml` other than the example). Store Wi-Fi and access-point passwords with `!secret` as well. **Never commit camera RTSP URLs containing credentials, live passwords, tokens, or private local configuration.** Git history retains previously committed secrets even if later deleted.

### 3. Validate and flash

```bash
esphome config panel.yaml
esphome run panel.yaml
```

For the initial flash, connect the board via USB and select the appropriate serial port when prompted. Once the device is on Wi-Fi, subsequent updates can use ESPHome OTA where network discovery/connectivity permits.

For serial logs:

```bash
esphome logs panel.yaml
```

The first build needs internet access to fetch dependencies, Google Fonts, and the **compile-time RainViewer basemap**. After boot, live data and radar still require internet access; this is not an offline weather receiver.

## Data sources and refresh rates

| Data | Provider | Refresh |
|---|---|---|
| Current conditions | [Open-Meteo](https://open-meteo.com/) | 10 minutes |
| AQI / PM2.5 / ozone / UV | [Open-Meteo Air Quality](https://open-meteo.com/en/docs/air-quality-api) | 15 minutes |
| Forecast | [National Weather Service](https://www.weather.gov/documentation/services-web-api) | 30 minutes |
| Active alerts | [National Weather Service](https://www.weather.gov/documentation/services-web-api) | 2 minutes |
| Historical radar frames | [RainViewer](https://www.rainviewer.com/api.html) | 5 minutes |
| NWS forecast-grid URL | NWS `/points` endpoint | Resolved on startup and again every 6 hours |

The NWS endpoints are specific to US locations. Other locations need an alternative forecast/alert source or changes to those pages. Rates above describe the configured scheduling; network failures, retries, and API availability can affect when data actually updates.

Runtime weather/radar network access includes `api.open-meteo.com`, `air-quality-api.open-meteo.com`, `api.weather.gov`, `api.rainviewer.com`, RainViewer's tile host, and an NTP server. Camera functionality additionally needs **LAN reachability to the configured Frigate server on TCP port 5000**. HTTPS certificate validation is enabled, and the panel waits for a valid SNTP clock before starting its initial API requests.

## Cameras (optional, standalone from Home Assistant)

The fifth page reads **latest-frame JPEGs from Frigate's HTTP API**, not directly from camera RTSP streams or go2rtc's MJPEG endpoint. **Frigate must already be running**; the panel does not host or decode H.264 video.

Camera names are defined in the `refresh_camera_frame` script in [`panel.yaml`](panel.yaml):

| Display label | Frigate camera name |
|---|---|
| CAMERA 07C3 | `camera_07c3` |
| CAMERA 106874268 | `camera_106874268` |
| BACKYARD (default) | `backyard` |
| UTILITIES INDOOR | `cam_utilities_indoor` |
| USB WEBCAM | `usb_webcam` |

These names **must match keys under `cameras:` in your own Frigate configuration**. Modify both the camera names and matching display labels in the script if your installation differs. The panel starts with Backyard selected.

The API path for an individual camera is:

```text
http://FRIGATE_SERVER_IP:5000/api/CAMERA_NAME/latest.jpg
```

Check one image **from a machine that can reach your Frigate server**, before flashing the ESP32:

```bash
curl --fail --show-error --location --max-time 15 \
  "http://FRIGATE_SERVER_IP:5000/api/backyard/latest.jpg" \
  --output /tmp/frigate-backyard.jpg
file /tmp/frigate-backyard.jpg
```

The first command should succeed and the second should identify a JPEG. Then verify the ESP32 can reach the same host and port over its Wi-Fi network. If port 5000 is not published or is blocked, the camera page will not work until the ESP32 has an authorized network path to the API.

**Security:** Frigate's port **5000 is unauthenticated** and intended for trusted internal integrations. Do **not** expose it to the public internet or an untrusted LAN; restrict access using your network/firewall configuration. Frigate's port 8971 is authenticated, but the example firmware does **not** implement Frigate's login flow and currently uses port 5000. See [Frigate's authentication documentation](https://docs.frigate.video/configuration/authentication/).

### Refresh behavior and limitations

- On entry, the camera page requests the selected camera's latest JPEG. Previous/Next switches the camera.
- A one-second timer requests a new image **only while the camera page is visible** and only when a previous request is not pending. Actual update speed varies with Wi-Fi, image downloads, and JPEG decoding; **this is not 1 FPS guaranteed video**.
- Only one camera image buffer is allocated (`448×336` RGB565, about 294 KiB of pixel data), separate from the existing radar buffers.
- Requests/decoding failures display `CAMERA UNAVAILABLE`. Weather, radar, and the other pages remain independent.
- These are the latest frames from Frigate's **detect pipeline**, not archived object-event snapshots and not necessarily the recording stream's full resolution. Frigate can provide its last frame or a preview fallback while a camera is offline.
- The extra `*_panel` MJPEG streams suggested in earlier versions of this setup are **not used by the current firmware**. They may be removed from Frigate if nothing else depends on them.

This camera feature has been checked against the configuration and current Frigate/ESPHome API documentation; its performance and ESPHome build compatibility have **not yet been verified on the physical board**.

## MQTT and Home Assistant

This project intentionally has **no ESPHome native `api:` component** and does not depend on Home Assistant.

The checked-in configuration includes an MQTT client with discovery disabled, `reboot_timeout: 0s`, and a `!secret mqtt_broker` address. MQTT is **not required for weather, radar, or camera images**. You can remove the MQTT block if you prefer Wi-Fi/HTTP only. If you keep it, set `mqtt_broker` in your ignored `secrets.yaml`.

## Radar implementation

RainViewer's public metadata endpoint supplies the available historical frame paths. The panel selects the latest four, creates their centered 512-pixel radar image URLs, and requests them sequentially.

The map is embedded in the firmware at compile time. Online radar frames are decoded into RGB565 images at 480×480. During a refresh, the existing animation is stopped and hidden, the success mask is reset, and new image buffers are downloaded. LVGL is given the four new sources only when the success mask confirms all downloads. If any frame fails, the UI displays a radar update error instead of starting an animation with an invalid source.

Four 480×480 RGB565 images require approximately **1.76 MiB** for the pixel data alone (before transparency, other image buffers, LVGL, and networking overhead). Working PSRAM is important.

## Simulator and screenshots

The repository also contains a host/SDL LVGL simulator **for the four original weather pages**:

- [`simulator/simulator.yaml`](simulator/simulator.yaml) — similar four-page layout with deterministic sample weather/alert data.
- [`.github/workflows/render-screenshots.yml`](.github/workflows/render-screenshots.yml) — builds the simulator and generates screenshot files.
- [`docs/screenshots/`](docs/screenshots/) — the images shown above.

To regenerate screenshots, run the **Render LVGL screenshots** workflow manually in GitHub Actions, or change the simulator/workflow paths that trigger it. The workflow retrieves map/radar assets, runs the simulator in a virtual X display, captures all four pages, and commits changed screenshots. Its simulated location and values are independent from the live physical panel. The Cameras page does not currently have a simulator screenshot.

## Troubleshooting

| Symptom | What to check |
|---|---|
| `lv_image_set_src: failed to get image info` | Make sure the `animimg` starts with a valid embedded image and that the four online images are only attached after successful downloads. Check the radar download error logs. |
| Radar shows `RADAR UPDATE ERROR` | Check internet/DNS/HTTPS access, RainViewer metadata, image URLs, and available PSRAM. One failed frame prevents the new animation set from being activated. |
| Radar appears geographically offset | Ensure `basemap_zoom` and `radar_zoom` match and latitude/longitude substitutions are correct. |
| Forecast missing | Check NWS `/points` resolution, forecast HTTP status, TLS, and whether the location is covered by NWS. |
| Clock missing or HTTPS failures on startup | Check Wi-Fi, DNS, and SNTP time synchronization; TLS validation needs a correct clock. |
| Device reports allocation/decoding errors | Check octal PSRAM settings, memory usage, and downloaded image formats. |
| Touch or screen not working | Confirm the exact board revision and the ST7701S, GT911, RGB, SPI, and backlight GPIO assignments. |
| `esphome config` fails on image options | The device YAML uses a legacy image schema; check ESPHome version compatibility before changing unrelated hardware pins. |
| All cameras show `CAMERA UNAVAILABLE` | Verify `frigate_host` points to **Frigate**, not a camera or the MQTT broker. From a LAN computer, run the `curl` JPEG test above; then check whether port 5000 is reachable from the ESP32's Wi-Fi. Inspect ESPHome logs for HTTP or JPEG decoder errors. |
| One camera shows `CAMERA UNAVAILABLE` | Verify its Frigate camera name, that its detection pipeline is producing frames, and that `/api/CAMERA_NAME/latest.jpg` returns a JPEG. |
| Camera request returns 401/403 or a login page | You likely contacted an authenticated proxy/UI, not an accessible internal API. The demo firmware does not authenticate to Frigate; keep port 5000 access limited to trusted devices rather than exposing it broadly. |
| Camera image is stale or low resolution | The latest-frame API uses Frigate's detection frame; check the source's detect input, resolution, and status. This is not RTSP video playback. |
| Camera page does not work while weather works | Internet access for weather and local access to Frigate are separate network requirements. Check your VLAN/firewall rules and Frigate port mapping. |

Use `esphome logs panel.yaml` to inspect request status, connection errors, and LVGL/image decoder warnings.

## Repository contents

```text
.
├── panel.yaml                            # Physical ESP32-S3 firmware + camera page
├── secrets.example.yaml                  # Non-secret placeholders for LAN hosts
├── .gitignore                            # Excludes local secrets and builds
├── simulator/
│   └── simulator.yaml                    # Host/SDL documentation simulator
├── docs/
│   └── screenshots/                      # Rendered four-page previews
├── .github/workflows/
│   └── render-screenshots.yml            # Screenshot generator
└── LICENSE
```

## License

[GNU General Public License v3.0](LICENSE).
