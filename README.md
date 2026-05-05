# NanoC6 Noctua PWM Controller

This repository configures an M5Stack NanoC6 to control one Noctua NF-A8 5V PWM fan with ESPHome.

## Wiring

Use the NanoC6 Grove port for one fan:

| NanoC6 Grove | NanoC6 signal | Noctua wire | Purpose |
| --- | --- | --- | --- |
| Black | GND | Black | Shared ground |
| Red | 5 V | Yellow | Fan power |
| Yellow | GPIO2 / G2 | Blue | PWM control |
| White | GPIO1 / G1 | Green | Optional RPM feedback |

The RPM wire is optional. If connected, use NanoC6 GPIO1 / G1 with the internal pull-up configured in `computer-cooling-fan.yaml`.

Do not connect this fan to a normal PC motherboard fan header. This is the 5 V Noctua variant, not the 12 V PC fan-header variant.

## Secrets

`secrets.yaml` is SOPS-encrypted. The compile wrapper decrypts it into environment variables only; it does not write a decrypted secrets file.

- `wifi.ssid`
- `wifi.password`
- `wifi.ap_ssid`
- `wifi.ap_password`
- `api_encryption_key`
- `ota_password`
- `fan_http.user`
- `fan_http.password`

Do not commit real secrets.

Compile through the wrapper:

```bash
./compile-firmware
```

## Control Paths

Home Assistant uses the encrypted ESPHome native API. Scripts use the ESPHome HTTP web server API. The script path is separate from Home Assistant and the native API, so script calls do not create native API client connect or disconnect events.

UDP is not used.

The fan boots to 60%. If Home Assistant or any other state-subscribing native API client disconnects and no state-subscribing native API client remains after the delay, the fan falls back to 60%.

If the configured Wi-Fi network is unavailable, ESPHome enables its fallback AP using `wifi.ap_ssid` and `wifi.ap_password`.

## Speed Mapping

`computer-cooling-fan.yaml` uses `speed_count: 5`, which gives five nonzero speed levels plus off:

| Requested speed | HTTP speed level | PWM duty |
| ---: | ---: | ---: |
| 0% | off | 0% |
| 20% | 1 | 20% |
| 40% | 2 | 40% |
| 60% | 3 | 60% |
| 80% | 4 | 80% |
| 100% | 5 | 100% |

Boot speed is 60%, which is `speed: 3`. HA-loss fallback speed is also 60%, which is `speed: 3`.

## HTTP Usage

Use POST for speed changes:

| Speed | Endpoint |
| ---: | --- |
| 0% | `POST /fan/case_fan/turn_off` |
| 20% | `POST /fan/case_fan/turn_on?speed_level=1` |
| 40% | `POST /fan/case_fan/turn_on?speed_level=2` |
| 60% | `POST /fan/case_fan/turn_on?speed_level=3` |
| 80% | `POST /fan/case_fan/turn_on?speed_level=4` |
| 100% | `POST /fan/case_fan/turn_on?speed_level=5` |

The `set-case-fan` wrapper uses only HTTP POST:

```bash
export FAN_HOST='computer-cooling-fan.local'
export FAN_USER='fan'
export FAN_PASS='CHANGE_ME_HTTP_PASSWORD'

./set-case-fan 60
```

The wrapper accepts only `0`, `20`, `40`, `60`, `80`, and `100`. It does not use ESPHome native API, Home Assistant, or UDP.

## Narrow Tests

Without contacting the device, check the repository artifacts:

```bash
bash -n ./set-case-fan
bash -n ./compile-firmware
rg 'speed: 3|speed_count: 5|GPIO2|25000 Hz|GPIO1' computer-cooling-fan.yaml
rg -i 'POST /fan/case_fan|boot.*60|fallback.*60|native API|UDP' README.md
```

After the device has already been flashed and is on the trusted LAN, a live state read is:

```bash
curl --user "$FAN_USER:$FAN_PASS" \
  "http://${FAN_HOST:-computer-cooling-fan.local}/fan/case_fan"
```

Live speed changes must use POST through `set-case-fan` or the endpoints above.
