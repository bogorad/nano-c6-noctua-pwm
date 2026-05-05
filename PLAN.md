Below is a complete design for the **Noctua NF-A8 5V PWM + M5Stack NanoC6 + ESPHome** controller with:

```text
HA path:     ESPHome native API, optional
Script path: HTTP REST API, separate from HA
Fan speeds: 0 / 20 / 40 / 60 / 80 / 100
Boot:        60%
HA loss:     60%
UDP:         not used
```

## 1. Hardware conclusion

The **M5Stack NanoC6 is sufficient for one NF-A8 5V PWM fan**.

The fan is rated for **5 V**, **0.15 A max**, **0.75 W max**, **4 V starting voltage**, and **5.5 V max operating voltage**. Its declared speed points are **0 rpm at 0% PWM**, **400 rpm at 20% PWM**, and **2200 rpm max**. ([Noctua][1])

The NanoC6 Grove port exposes **5 V**, **GND**, **G1**, and **G2**, and M5Stack documents the Grove output as **5 V / 600 mA max**, dependent on USB input. The board itself is an ESP32-C6FH4 device with 2.4 GHz Wi-Fi 6 support. ([M5Stack Docs][2])

Power estimate:

```text
NanoC6 Wi-Fi:     ~106 mA class, from M5Stack spec
NF-A8 5V PWM:      150 mA max
Total:             ~256 mA
Grove 5 V limit:   600 mA max
```

Use a decent USB-C supply. A **5 V / 1 A or better** phone charger is plenty for one fan. Avoid relying on an unknown weak USB port.

## 2. Wiring

Noctua’s 4-wire 5 V PWM fan uses:

```text
Black  = GND
Yellow = +5 V
Green  = RPM / tach
Blue   = PWM control
```

Noctua’s own page shows the 4-pin PWM fan mapping as black ground, yellow voltage, green RPM signal, and blue PWM signal. The table text on the page has a likely typo where pin 4 says “Black: PWM Signal”; the diagram and Noctua’s standard wire colors identify **blue** as PWM. ([Noctua][1])

NanoC6 Grove port:

```text
Black  = GND
Red    = 5 V
Yellow = G2 / GPIO2
White  = G1 / GPIO1
```

Wire it like this:

| NanoC6 Grove | NanoC6 signal | Noctua wire | Purpose               |
| ------------ | ------------: | ----------- | --------------------- |
| Black        |           GND | Black       | Shared ground         |
| Red          |           5 V | Yellow      | Fan power             |
| Yellow       |    GPIO2 / G2 | Blue        | PWM control           |
| White        |    GPIO1 / G1 | Green       | Optional RPM feedback |

Minimal version:

```text
NanoC6 red    -> Noctua yellow
NanoC6 black  -> Noctua black
NanoC6 G2     -> Noctua blue
Noctua green  -> leave disconnected unless you want RPM
```

Do **not** connect the fan to a normal PC motherboard fan header. This is the **5 V** Noctua variant, and Noctua labels it “not for use in PCs.” ([Noctua][1])

## 3. PWM rules

Noctua is explicit: control the fan through the **PWM pin**, not by PWM-switching the power line. They warn that PWM over the power line can damage the fan. ([Noctua][3])

Use:

```text
PWM frequency: 25 kHz
Accepted range: 21–28 kHz
Logic high: 3.3 V or 5 V accepted
Logic low: below 0.8 V
```

Noctua says 3.3 V logic is recognized as high, so the ESP32-C6 GPIO can drive the fan’s PWM input directly. No MOSFET, transistor, or level shifter is needed for the PWM line. ([Noctua][3])

Also important:

```text
No PWM signal connected -> fan runs full speed
0% PWM signal           -> this model stops
<20% duty               -> undefined by Noctua
```

That matches your requested steps well: **0, 20, 40, 60, 80, 100**. No intermediate “bad zone” is needed. ([Noctua][3])

## 4. Control design

Use two independent paths.

```text
Path A: HA / native API
  - ESPHome native API
  - encrypted
  - HA may connect/disconnect
  - if HA disconnects and no state-subscribing API client remains, fan goes to 60%

Path B: script / HTTP
  - ESPHome web_server REST API
  - no UDP
  - script sends HTTP POST
  - does not touch native API
  - no connect/disconnect semantics
```

ESPHome’s native API supports `on_client_connected` and `on_client_disconnected` triggers. The API component also has an `api.connected` condition with `state_subscription_only: true`, specifically to distinguish real state-subscribing clients such as Home Assistant from logger-only connections like `esphome logs`. ([ESPHome - Smart Home Made Simple][4])

ESPHome’s web server exposes a browser UI and a REST API. The web API uses this URL shape:

```text
/<domain>/<entity_name>[/<action>?<param>=<value>]
```

For fans, it supports:

```text
GET  /fan/<entity_name>
POST /fan/<entity_name>/turn_on?speed_level=N
POST /fan/<entity_name>/turn_off
POST /fan/<entity_name>/toggle
```

The fan API reports `speed_level`, and `speed_level` ranges from `1` to the maximum supported fan speed level. ([ESPHome - Smart Home Made Simple][5])

## 5. Speed mapping

ESPHome’s speed fan supports `speed_count`; ESPHome documents that `speed_count: 2` gives 50% and 100%, while `speed_count: 100` gives 1% increments. ([new.esphome.io][6])

Use:

```yaml
speed_count: 5
```

That gives five nonzero levels plus off:

| Desired | HTTP/native speed level | PWM duty |
| ------: | ----------------------: | -------: |
|      0% |                     off |       0% |
|     20% |                       1 |      20% |
|     40% |                       2 |      40% |
|     60% |                       3 |      60% |
|     80% |                       4 |      80% |
|    100% |                       5 |     100% |

## 6. Full ESPHome YAML

This version keeps the HTTP entity name simple: `case_fan`.

```yaml
substitutions:
  device_name: computer-cooling-fan
  friendly_name: Computer Cooling Fan

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}

  # Boot to the intended normal cooling speed.
  on_boot:
    priority: -100
    then:
      - fan.turn_on:
          id: case_fan
          speed: 3   # 60%

esp32:
  # If your ESPHome install knows the NanoC6 board name, you can replace this
  # with a more specific board value. The important part is esp32c6 + esp-idf.
  variant: esp32c6
  flash_size: 4MB
  framework:
    type: esp-idf

logger:

api:
  id: native_api
  reboot_timeout: 0s
  encryption:
    key: !secret api_encryption_key

  # This fires for native API clients. That includes HA, ESPHome dashboard/logs,
  # or any native API script. Your script path should use HTTP instead.
  on_client_connected:
    then:
      - logger.log:
          format: "Native API client connected: %s from %s"
          args: ["client_info.c_str()", "client_address.c_str()"]

  # On any native API disconnect, wait briefly. If no state-subscribing client
  # remains, treat it as HA gone and fall back to 60%.
  on_client_disconnected:
    then:
      - logger.log:
          format: "Native API client disconnected: %s from %s"
          args: ["client_info.c_str()", "client_address.c_str()"]
      - delay: 10s
      - if:
          condition:
            lambda: |-
              return !id(native_api).is_connected(true);
          then:
            - logger.log: "No state-subscribing native API client remains; setting fan to 60%"
            - fan.turn_on:
                id: case_fan
                speed: 3   # 60%

ota:
  - platform: esphome

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: none

  # Useful for recovery. Remove if you do not want fallback AP mode.
  ap:
    ssid: "${friendly_name} Fallback"
    password: !secret fallback_ap_password

captive_portal:

# HTTP control path for scripts.
# This is independent of the native API used by HA.
web_server:
  port: 80
  version: 3
  auth:
    username: !secret fan_http_user
    password: !secret fan_http_password

output:
  - platform: ledc
    id: noctua_pwm
    pin: GPIO2
    frequency: 25000 Hz
    inverted: false

fan:
  - platform: speed
    id: case_fan
    name: "case_fan"
    output: noctua_pwm
    speed_count: 5
    restore_mode: ALWAYS_ON

# Optional: RPM monitoring.
# Wire Noctua green tach wire to NanoC6 G1 / GPIO1.
sensor:
  - platform: pulse_counter
    id: fan_rpm
    name: "case_fan_rpm"
    pin:
      number: GPIO1
      inverted: true
      mode:
        input: true
        pullup: true
    unit_of_measurement: "RPM"
    update_interval: 5s
    filters:
      # Noctua tach is two pulses per revolution.
      # ESPHome pulse_counter reports pulses/minute, so divide by 2.
      - multiply: 0.5
    count_mode:
      rising_edge: INCREMENT
      falling_edge: DISABLE
```

ESPHome requires ESP-IDF for ESP32-C6 variants; Arduino framework is not available for ESP32-C6 in ESPHome. ([ESPHome - Smart Home Made Simple][7]) ESPHome’s LEDC output exposes ESP32 PWM, supports a `frequency` setting, and uses a pin plus ID as shown above. ([ESPHome - Smart Home Made Simple][8])

The `api.reboot_timeout: 0s` is deliberate. ESPHome’s default API behavior is to reboot after 15 minutes without an API client; setting it to `0s` disables that. For a cooling controller, periodic reboot because HA is down is not desirable. ([ESPHome - Smart Home Made Simple][4])

## 7. Secrets file

Example `secrets.yaml`:

```yaml
wifi_ssid: "YOUR_WIFI"
wifi_password: "YOUR_WIFI_PASSWORD"

fallback_ap_password: "CHANGE_ME_RECOVERY_PASSWORD"

api_encryption_key: "YOUR_ESPHOME_API_KEY"

fan_http_user: "fan"
fan_http_password: "CHANGE_ME_HTTP_PASSWORD"
```

The web server supports simple Digest authentication with username/password. ([ESPHome - Smart Home Made Simple][9])

Do not expose this HTTP interface to the Internet. The ESPHome web server is fine for a trusted LAN, WireGuard, Tailscale, CF-One/WARP-style private access, etc. It is not a public-facing API service.

## 8. HTTP script endpoints

Because the fan entity is named:

```yaml
name: "case_fan"
```

the REST paths are simple.

### Read current state

```bash
curl -u "$FAN_USER:$FAN_PASS" \
  "http://computer-cooling-fan.local/fan/case_fan"
```

Expected shape includes state and speed level:

```json
{
  "id": "fan/case_fan",
  "state": "ON",
  "value": true,
  "speed_level": 3
}
```

ESPHome documents `GET /fan/<entity_name>` as the fan state endpoint and `speed_level` as the supported speed value. ([ESPHome - Smart Home Made Simple][5])

### Set 0%

```bash
curl -X POST -u "$FAN_USER:$FAN_PASS" \
  "http://computer-cooling-fan.local/fan/case_fan/turn_off"
```

### Set 20%

```bash
curl -X POST -u "$FAN_USER:$FAN_PASS" \
  "http://computer-cooling-fan.local/fan/case_fan/turn_on?speed_level=1"
```

### Set 40%

```bash
curl -X POST -u "$FAN_USER:$FAN_PASS" \
  "http://computer-cooling-fan.local/fan/case_fan/turn_on?speed_level=2"
```

### Set 60%

```bash
curl -X POST -u "$FAN_USER:$FAN_PASS" \
  "http://computer-cooling-fan.local/fan/case_fan/turn_on?speed_level=3"
```

### Set 80%

```bash
curl -X POST -u "$FAN_USER:$FAN_PASS" \
  "http://computer-cooling-fan.local/fan/case_fan/turn_on?speed_level=4"
```

### Set 100%

```bash
curl -X POST -u "$FAN_USER:$FAN_PASS" \
  "http://computer-cooling-fan.local/fan/case_fan/turn_on?speed_level=5"
```

## 9. One wrapper script

`set-case-fan`:

```bash
#!/usr/bin/env bash
set -euo pipefail

host="${FAN_HOST:-computer-cooling-fan.local}"
user="${FAN_USER:-fan}"
pass="${FAN_PASS:?set FAN_PASS}"
speed="${1:?usage: set-case-fan 0|20|40|60|80|100}"

case "$speed" in
  0)
    path="/fan/case_fan/turn_off"
    ;;
  20)
    path="/fan/case_fan/turn_on?speed_level=1"
    ;;
  40)
    path="/fan/case_fan/turn_on?speed_level=2"
    ;;
  60)
    path="/fan/case_fan/turn_on?speed_level=3"
    ;;
  80)
    path="/fan/case_fan/turn_on?speed_level=4"
    ;;
  100)
    path="/fan/case_fan/turn_on?speed_level=5"
    ;;
  *)
    echo "Invalid speed: $speed" >&2
    echo "Allowed: 0 20 40 60 80 100" >&2
    exit 2
    ;;
esac

curl --fail --silent --show-error \
  --request POST \
  --user "$user:$pass" \
  "http://${host}${path}"

echo "fan speed set to ${speed}%"
```

Usage:

```bash
export FAN_PASS='CHANGE_ME_HTTP_PASSWORD'

set-case-fan 100
set-case-fan 60
set-case-fan 0
```

This script never touches the ESPHome native API, so it will not trigger the HA disconnect behavior.

## 10. HA behavior

HA can still use the normal ESPHome integration through the native API.

Behavior with the YAML above:

```text
Device boots:
  fan -> 60%

HA connects:
  no speed change

HA changes speed:
  normal HA fan control works

HA disconnects:
  wait 10 seconds
  if no state-subscribing native API client remains:
    fan -> 60%

HTTP script sets speed:
  no native API connection involved
  no HA disconnect logic involved
```

The 10-second delay is intentional. It avoids false fallback during HA reconnects, ESPHome dashboard interactions, and short transient API disconnects.

## 11. Optional RPM feedback

If you connect the Noctua green wire to GPIO1, the YAML above reports RPM.

Noctua says the fan tach/RPM signal uses **two pulses per revolution**, so the RPM formula is:

```text
RPM = pulses_per_minute / 2
```

Noctua also says the tach line needs a pull-up to the controller’s suitable voltage. ([Noctua][3]) The ESPHome `pulse_counter` docs show using an internal pull-up on a GPIO input, which is what the YAML does. ([ESPHome - Smart Home Made Simple][10])

Do not pull the tach wire up to 5 V when reading it with the ESP32-C6 GPIO. Use the internal 3.3 V pull-up as shown.

## 12. Test procedure

Before mounting it on the computer:

1. Flash the NanoC6.
2. Connect only:

   ```text
   Noctua black  -> NanoC6 GND
   Noctua yellow -> NanoC6 5 V
   Noctua blue   -> NanoC6 GPIO2 / G2
   ```
3. Power NanoC6 from USB.
4. Fan should start at **60%** after boot.
5. Test HTTP state:

   ```bash
   curl -u fan:'CHANGE_ME_HTTP_PASSWORD' \
     "http://computer-cooling-fan.local/fan/case_fan"
   ```
6. Test 60%:

   ```bash
   curl -X POST -u fan:'CHANGE_ME_HTTP_PASSWORD' \
     "http://computer-cooling-fan.local/fan/case_fan/turn_on?speed_level=3"
   ```
7. Test off:

   ```bash
   curl -X POST -u fan:'CHANGE_ME_HTTP_PASSWORD' \
     "http://computer-cooling-fan.local/fan/case_fan/turn_off"
   ```
8. Test fail-safe:

   ```bash
   curl -X POST -u fan:'CHANGE_ME_HTTP_PASSWORD' \
     "http://computer-cooling-fan.local/fan/case_fan/turn_on?speed_level=5"
   ```
9. Only after confirming control works, mount it on the computer.

## 13. Practical notes

Use **HTTP POST**, not GET, for speed changes. ESPHome documents fan control as POST actions. ([ESPHome - Smart Home Made Simple][5])

Use **static DHCP lease** for the NanoC6 if this controls actual cooling. mDNS is convenient, but a fixed address is better for scripts.

Do not use 1–19% duty. Noctua says sub-20% duty is undefined, and the NF-A8 5V PWM explicitly documents 20% as 400 rpm. ([Noctua][3])

For one fan, powering from NanoC6 Grove 5 V is fine. For multiple fans, use a separate 5 V supply and tie grounds together. Noctua explicitly says that when using an external fan power supply, the fan ground and controller ground must be connected as a common reference for PWM. ([Noctua][3])

Final recommended topology:

```text
USB-C 5 V supply
   |
   v
M5Stack NanoC6
   |-- Grove 5 V  -> Noctua yellow
   |-- Grove GND  -> Noctua black
   |-- GPIO2/G2   -> Noctua blue PWM
   '-- GPIO1/G1   -> Noctua green RPM, optional

HA:
   ESPHome native API, encrypted, optional

Scripts:
   HTTP POST to ESPHome web_server REST API
```

[1]: https://www.noctua.at/en/products/nf-a8-5v-pwm/specifications?utm_source=chatgpt.com "NF-A8 5V PWM: Specifications"
[2]: https://docs.m5stack.com/en/core/M5NanoC6 "m5-docs"
[3]: https://www.noctua.at/en/support/faqs/microcontroller-guide-pwm-setup-and-rpm-monitoring "FAQ: Microcontroller guide: PWM setup and RPM monitoring of your Noctua fan | Noctua"
[4]: https://esphome.io/components/api/ "Native API Component - ESPHome - Smart Home Made Simple"
[5]: https://esphome.io/web-api/ "Web Server API - ESPHome - Smart Home Made Simple"
[6]: https://new.esphome.io/components/fan/speed/ "Speed Fan - ESPHome - Smart Home Made Simple"
[7]: https://esphome.io/components/esp32/ "ESP32 Platform - ESPHome - Smart Home Made Simple"
[8]: https://esphome.io/components/output/ledc/ "ESP32 LEDC Output - ESPHome - Smart Home Made Simple"
[9]: https://esphome.io/components/web_server/ "Web Server Component - ESPHome - Smart Home Made Simple"
[10]: https://esphome.io/components/sensor/pulse_counter/?utm_source=chatgpt.com "Pulse Counter Sensor"
