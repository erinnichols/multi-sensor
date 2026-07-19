# multi-sensor

A small ESP32-C3 board that answers two questions at once: *"is someone in here?"* and *"what's it like in here?"* — motion + temperature/humidity on a single perfboard, built almost entirely out of parts that had been sitting in a drawer since an Adafruit IoT kit impulse-buy several years ago.

## Why this exists

I was disappointed in the Zooz 4-in-1 sensor I'd been using in my bedroom to turn on lights immediately (rather than waiting for mmwave to sense a person) because it wasn't great at detecting humidity. And since getting out of the shower triggered my smoke alarm a couple of months ago, I've been on a quest to deal with my DC rowhouse top floor's warm season humidity. As usual, one project begets many other projects. Plus, I had the parts sitting around and felt the need to solder something.

## Hardware

| Ref | Part | Notes |
|-----|------|-------|
| U1 | ESP32-C3 SuperMini | The brains |
| J1 | AHT20 (Adafruit #5183) | I2C temp/humidity |
| J2 | PIR motion sensor (Adafruit #189) | Digital motion output |
| R1, R2 | 4.7kΩ resistors | I2C pull-ups (SDA / SCL) |
| C1 | 100nF ceramic capacitor | Decoupling on AHT20 VCC |

## Schematic

![multi-sensor schematic](multi-sensor.svg)

## Wiring

- **I2C bus:** SDA on GPIO0, SCL on GPIO1, each with a 4.7kΩ pull-up to 3.3V (R1, R2). The AHT20 breakout doesn't have onboard pull-ups, so these are load-bearing.
- **Decoupling:** C1 (100nF) sits across AHT20 VCC/GND, close to the sensor - standard "don't let the I2C transaction glitch on you" insurance.
- **PIR:** OUT on GPIO3, power off the 5V rail rather than 3.3V. The Adafruit 189 has an internal voltage regulator, so it wants 5V, and its digital output is 3.3V-logic-friendly regardless, so it plays nicely straight into the C3's GPIO.

## ESPHome config

```yaml
# Board: ESP32-C3 Super Mini (Generic)
# Definition: definitions/boards/esp32-c3-supermini/manifest.yaml

esphome:
  name: bedroom-multisensor
  friendly_name: Bedroom Multisensor

esp32:
  variant: esp32c3
  flash_size: 4MB
  framework:
    type: esp-idf

logger:

api:
  encryption:
    key: !secret multisensor_encryption

ota:
  - platform: esphome

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: Bedroom Multise Fallback Hotspot
    password: !secret multisensor_fallback

captive_portal:

# I2C bus for the AHT20
i2c:
  sda: GPIO0
  scl: GPIO1
  scan: true

sensor:
  - platform: aht10
    variant: AHT20
    temperature:
      name: "Bedroom Temperature"
      id: bedroom_temp
      filters:
        - offset: -1.54
    humidity:
      name: "Bedroom Humidity"
      id: bedroom_humidity
    update_interval: 15s

binary_sensor:
  - platform: gpio
    pin:
      number: GPIO3
      mode: INPUT
    name: "Bedroom PIR Motion"
    device_class: motion
    filters:
      - delayed_on: 200ms
      - delayed_off: 10s
```

A few notes on the config:

- `scan: true` on the I2C bus is handy the first time you flash this — check the logs for the AHT20 showing up at `0x38` before you go chasing phantom wiring bugs.
- The `-1.54` temperature offset exists because the AHT20 reads a little warm sitting this close to the ESP32-C3 — measured against a reference and dialed in.
- `delayed_on: 200ms` debounces the PIR's leading edge; `delayed_off: 10s` keeps it from flapping off the instant motion technically stops. Same "cheap PIRs chatter" reasoning as any other motion sensor in the house.

## Home Assistant

Once flashed, this shows up as three entities via native ESPHome integration:
- `binary_sensor.bedroom_multisensor_bedroom_pir_motion`
- `sensor.bedroom_multisensor_bedroom_temperature`
- `sensor.bedroom_multisensor_bedroom_humidity`

No custom component needed — ESPHome's native API handles discovery. From here it slots into whatever room-presence template sensor logic already exists, same as every other motion source in the house.

## Known gotchas / lessons

- **Pull-ups are not optional.** Skip R1/R2 and the AHT20 will either not show up in the I2C scan or return garbage readings intermittently — looks like a flaky sensor, isn't.
- **PIR warm-up time.** Give it 30-60 seconds after power-on before trusting the output; PIRs need a settling period and will happily report false motion during it.

## Future ideas

- Case/enclosure (currently living the exposed-perfboard lifestyle)
- Maybe an FSR-style occupancy confidence blend if this ends up living somewhere presence-detection-critical

## License

MIT — build one, wire it wrong the first time like tradition demands, fix it, ship it.