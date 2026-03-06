# modbus_slave_tcp

ESPHome component that turns an **ESP32** into a **Modbus TCP slave** using [Espressif esp-modbus](https://github.com/espressif/esp-modbus) on the **ESP-IDF** framework. Exposes coils, discrete inputs, input registers, and holding registers that you fill from sensors, GPIO, or lambdas.

**Tested on:** Ubuntu 24 with ESPHome 2026.1.5

## Requirements

- **ESP32** (tested with NodeMCU-32S).
- **ESP-IDF** framework (`framework: type: esp-idf` in your YAML).
- **WiFi** (TCP needs network; the component starts the slave after WiFi is connected).

## Installation

Add as an external component. From the same repo (local):

```yaml
external_components:
  - source:
      type: local
      path: components
    components: ["modbus_slave_tcp"]
    refresh: always
```

Or from GitHub:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/your-username/your-repo
      ref: main
    components: ["modbus_slave_tcp"]
```

## Configuration

```yaml
esp32:
  board: nodemcu-32s
  framework:
    type: esp-idf

modbus_slave_tcp:
  id: modbus_server
  port: 1502        # optional; default 1502
  slave_id: 1       # optional; default 1 (0–247)
  num_objects: 5    # optional; default 5; range 1–128 (coils, discrete inputs, input/holding registers each)
```

| Option       | Default | Description |
|-------------|---------|-------------|
| `port`      | 1502    | TCP port the slave listens on. |
| `slave_id`  | 1       | Modbus slave address (0–247). |
| `num_objects` | 5    | Size of each map: coils, discrete inputs, input registers, holding registers (each has `num_objects` elements, indices 0..num_objects-1). |

You can set `port` and `slave_id` in your YAML (e.g. `slave.yaml`); those values are used at runtime and override the library defaults.

No need to add `platformio_options` or build flags in your app YAML; the component injects esp-modbus, include paths, and the pre-build script.

## Usage

From **interval**, **automation**, or **lambda** (e.g. in a 1s interval), call the component by `id` to push values into the Modbus maps:

| Method | Description |
|--------|-------------|
| `id(modbus_server).set_coil(index, value)` | Set coil at `index` (0..num_objects-1). `value`: `true`/`false`. |
| `id(modbus_server).set_discrete_input(index, value)` | Set discrete input at `index`. Read-only for the master. |
| `id(modbus_server).set_input_register(index, value)` | Set input register at `index` (16-bit). Read-only for the master. |
| `id(modbus_server).set_holding_register(index, value)` | Set holding register at `index` (16-bit). Read/write for the master. |

Example: drive coils from a GPIO and fill holding/input registers from sensors:

```yaml
modbus_slave_tcp:
  id: modbus_server
  num_objects: 8

interval:
  - interval: 1s
    then:
      - lambda: |-
          id(modbus_server).set_coil(0, id(some_binary_sensor).state);
          id(modbus_server).set_holding_register(0, (uint16_t)id(some_sensor).state);
          id(modbus_server).set_input_register(0, (uint16_t)id(uptime_sensor).state);
```

## Full example

Complete `test1.yaml` using the component from GitHub, with GPIO for one coil, internal/WiFi/uptime sensors, and a 1s interval filling all four Modbus maps:

```yaml
esphome:
  name: mbsl1
  # modbus_slave_tcp from GitHub (git library)

esp32:
  board: nodemcu-32s
  framework:
    type: esp-idf

external_components:
  - source:
      type: git
      url: https://github.com/pintomax/esphome-modbus-slave-tcp
      ref: main
    components: ["modbus_slave_tcp"]
    refresh: always

modbus_slave_tcp:
  id: modbus_server
  port: 503        # optional; default in component (DEFAULT_MODBUS_PORT is 1502)
  slave_id: 1      # optional; default in component (DEFAULT_MODBUS_SLAVE_ID is 1), range 0–247
  num_objects: 5   # optional; default in component (DEFAULT_MODBUS_NUM_OBJECTS), range 1–128

# Optional: drive coils from GPIO or logic (call id(modbus_server).set_coil(index, true/false))
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO4
      inverted: true
      mode:
        input: true
        pullup: true
    id: coil1_source
    name: "Coil 1 from GPIO"

# Sensors used to fill holding/input registers
sensor:
  - platform: internal_temperature
    id: esp_temp
    name: "ESP internal temp"
    update_interval: 10s
    accuracy_decimals: 1
  - platform: wifi_signal
    id: wifi_rssi
    name: "WiFi RSSI"
    update_interval: 10s
  - platform: uptime
    id: uptime_sec
    name: "Uptime"
    update_interval: 1s

interval:
  - interval: 1s
    then:
      - lambda: |-
          // COILS (2)
          id(modbus_server).set_coil(0, true);
          id(modbus_server).set_coil(1, id(coil1_source).state);

          // HOLDING REGISTERS (3)
          id(modbus_server).set_holding_register(0, (uint16_t)id(esp_temp).state);  // integer °C
          id(modbus_server).set_holding_register(1, (uint16_t)(rand() % 101));  // fake humidity 0-100
          // WiFi signal 0-100%: RSSI -100 dBm = 0%, -30 dBm = 100%, clamped
          float rssi = id(wifi_rssi).state;
          float pct = (rssi + 100.0f) * 100.0f / 70.0f;
          if (pct < 0.0f) pct = 0.0f;
          if (pct > 100.0f) pct = 100.0f;
          id(modbus_server).set_holding_register(2, (uint16_t)pct);
          
          // INPUT REGISTERS (2)
          // Input reg 0 = uptime (seconds, wraps at 65535)
          id(modbus_server).set_input_register(0, (uint16_t)id(uptime_sec).state);
          // Input reg 1 = 88
          id(modbus_server).set_input_register(1, 88);

          // DISCRETE INPUTS (2)
          // Discrete inputs: 0=1 (ON), 1=0 (OFF)
          id(modbus_server).set_discrete_input(0, true);
          id(modbus_server).set_discrete_input(1, false);

# Wi-Fi is required for TCP (use secrets.yaml)
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

logger:
  level: INFO
```

Use a `secrets.yaml` with `wifi_ssid` and `wifi_password` in the same directory.

## Optional: request/response logging

The component uses a pre-build script that can patch esp-modbus to log each Modbus request and response (function code, address, count, length) at INFO level. By default this is **off**. To enable it, in the component set:

**File:** `components/modbus_slave_tcp/filter_esp_modbus.py`  
**Variable:** `ENABLE_MODBUS_REQUEST_RESPONSE_LOGGING = True`

Then run a normal compile. Set back to `False` and recompile to turn logging off again.

## References

- [Espressif esp-modbus](https://github.com/espressif/esp-modbus)
- [ESPHome external components](https://esphome.io/components/external_components.html)

## Maintainer

Massimiliano Pinto

massimiliano.pinto@gmail.com

https://github.com/pintomax
