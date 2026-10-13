# Zephyr v2 (internal name: Mistral)

**Thrust-vectored coaxial contra-rotating drone.** Two counter-rotating motors
lift the vehicle and cancel each other's torque. A two-servo gimbal vectors the
thrust for pitch and roll, and differential motor thrust controls yaw.

> **Status: flying.** Zephyr v2 has completed a full untethered flight: a hop off
> the ground, attitude held by the LQI through the climb and the hover, and a
> propulsive landing back down on its own thrust. No tether, no catch. Throttle
> was commanded by hand from the ground page; the controller flew the attitude.
> IMU (`imu`), attitude estimation (`attitude`: complementary filter + Fusion
> AHRS), the 6-state LQI (`lqr`), the DISARMED / ARMED / FLYING state machine,
> the TVC mixer, throttle and the WiFi ground station are implemented. The bench
> checklist in [`bench/VERIFY.md`](bench/VERIFY.md) still gates every flight. The
> barometer, optical flow, translation estimate and OLED display are still stubs,
> so there is no altitude or position feedback: height and translation are the
> pilot's job.

The full vehicle spec is [`PROJECT-CONTEXT.md`](PROJECT-CONTEXT.md).

The internal name survives in the identifiers, which are code and hardware facts
and stay as they are: `mistral_config.h`, the `mistral-` WiFi prefix, the
`mistral.send()` console helper.

## Hardware

| Subsystem | Component |
|-----------|-----------|
| Flight computer | Heltec WiFi LoRa 32 V3 (ESP32-S3) |
| IMU | ICM-42688-P |
| Barometer | BMP388 (display only) |
| Optical flow + range | MicoAir MTF-01P (UART) |
| Propulsion | 2× DZP30 motors, contra-rotating, 2 independent ESCs |
| TVC | 2× MG90S servos, gear-reduced (X 3:1, Y 4:1) |
| Display | On-board SSD1306 OLED |

## Project structure

```
zephyr-v2/
├── platformio.ini          build config (remaps into firmware/)
├── PROJECT-CONTEXT.md      authoritative vehicle spec
├── firmware/
│   ├── include/            mistral_config.h: every pin, limit and gain
│   ├── src/                main.cpp: wiring only
│   └── lib/                sensors (imu, bmp388, mtf01), fusion (attitude,
│                           translation), control (lqr), tvc, throttle,
│                           display, state_machine, safety, wifi_link, bench,
│                           FusionAhrs (vendored xioTechnologies/Fusion v1.3.3)
├── web/                    index.html: ground-station page, embedded into the firmware
├── bench/                  VERIFY.md: ordered bench verification checklist
└── docs/                   wiring diagram, actuators & comms, flight logs
    └── flight_logs/zephyr/  Zephyr (v1) CSV logs, August 2026
```

## Build & flash

Requires [PlatformIO](https://platformio.org/). The platform is pinned to the
pioarduino fork (Arduino-ESP32 core 3.x); PlatformIO fetches it on first build.

```bash
pio run                  # build
pio run -t upload        # flash over USB
pio device monitor       # serial monitor, 115200
```

## Ground station page

The vehicle hosts its own WiFi access point and serves the ground-station page:

1. Join the AP **`mistral-xxxx`**, where xxxx is the last four hex digits of the MAC, printed on
   serial at boot. The password is `cfg::WIFI_PASSWORD`.
2. Open **http://192.168.4.1**.

The page source is [`web/index.html`](web/index.html). The build embeds it into the
firmware, so edit that file and reflash. There is no second copy to keep in sync.
While joined to the AP you can also open the file straight from disk; it connects to
192.168.4.1. It has the flight-state bar, KILL, two-tap ARM and FLY, the throttle slider,
Attitude with both estimators and Zero IMU, the Nozzle readout, the Console, the live graph
stack, and Tuning for gains, servo trim and motor balance. Pack voltage, the
artificial-horizon SVG and trim copy-to-clipboard are still design only, described in
[`docs/actuators-comms.md`](docs/actuators-comms.md) §4. Record and Save write the flight to
an `.xlsx` with one row per telemetry frame.

## Flight logs

Zephyr (v1), the EDF predecessor, flown in August 2026. These links download the file:

| Log | Rows | Span | Contents |
|-----|------|------|----------|
| [`20260809-133551`](https://github.com/prabavac/Zephyr_v2/releases/download/logs-zephyr-2026-08/zephyr-flight-20260809-133551.csv) | 218 | 21.8 s | attitude only: IMU and nozzle pitch/roll |
| [`20260816-204453`](https://github.com/prabavac/Zephyr_v2/releases/download/logs-zephyr-2026-08/zephyr-flight-20260816-204453.csv) | 291 | 29.1 s | adds flow counts, velocity, position, standoff, translation command |
| [`20260817-193224`](https://github.com/prabavac/Zephyr_v2/releases/download/logs-zephyr-2026-08/zephyr-flight-20260817-193224.csv) | 252 | 25.3 s | as above plus flow surface quality |
| [`20260817-193357`](https://github.com/prabavac/Zephyr_v2/releases/download/logs-zephyr-2026-08/zephyr-flight-20260817-193357.csv) | 166 | 16.6 s | as above plus flow surface quality |

They are also in the tree under
[`docs/flight_logs/zephyr/`](docs/flight_logs/zephyr/), and attached to the
[`logs-zephyr-2026-08`](https://github.com/prabavac/Zephyr_v2/releases/tag/logs-zephyr-2026-08)
release, which is where the download links point. Standoff holds between 0.16 and 0.24 m
in all three flow logs, so these are low hovers, not free flight. Zephyr v2's own logs are
`.xlsx` from the Record button and are not published yet.

## Safety

- Verify every TVC servo sign on the test stand before any free flight.
- Bench-test motors with props off or the vehicle restrained.
- Throttle is software-gated behind arming; both ESCs sit at minimum otherwise.
- Losing the ground link for 1 s disarms. So does KILL, and hiding the page
  (switching apps or locking the phone).

## License

See [`LICENSE`](LICENSE).
