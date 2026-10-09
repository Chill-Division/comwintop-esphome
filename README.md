# ComWinTop RS485 sensors for ESPHome

ESPHome configs for reading ComWinTop (CWT) RS485 Modbus sensors with an M5Stack Unit PoE P4 (ESP32-P4), for grow-room monitoring in Home Assistant.

> [!WARNING]
> **These configs are not battle-tested.** Treat them as a starting point, not a finished product: they've had limited real-world use, and you may need to adjust register types, scaling or wiring for your exact sensor model and firmware. Check every reading against a reference instrument before using it to drive irrigation, dosing or alarms.

## Sensors

| Sensor | Measures | Baud | Supply |
| --- | --- | --- | --- |
| [THC-S substrate probe](esp32%20poe-p4%20comwintop%20thc-s%20substrate%20sensor.yaml) | Substrate moisture (VWC), temperature and bulk EC, plus an estimated pore-water EC | 4800 | 5–30 V DC |
| [CWT-BL-EC-4400-S water EC transmitter](esp32%20poe-p4%20comwintop%20cwt-bl-ec%20water%20ec%20sensor.yaml) | Water EC, 0–4400 µS/cm | 9600 | 12–24 V DC |
| [CWT-LEAF-TH leaf sensor](esp32%20poe-p4%20comwintop%20cwt-leaf-th%20leaf%20surface%20sensor.yaml) | Leaf wetness and leaf surface temperature | 4800 | 10–30 V DC |
| [CWT-PS PAR transmitter](esp32%20poe-p4%20comwintop%20cwt-ps%20par%20sensor.yaml) | PAR (PPFD), 0–2500 µmol/m²·s | 4800 | 7–30 V DC |
| [CWT-SWS-C wind speed sensor](esp32%20poe-p4%20comwintop%20cwt-sws-c%20wind%20speed%20sensor.yaml) | Wind speed (0–70 m/s) and 1-minute gust | 4800 | 10–30 V DC |
| [CWT-WLS water level sensor](esp32%20poe-p4%20comwintop%20cwt-wls%20water%20level%20sensor.yaml) | Water level (0–2 m range) and reservoir % full | 9600 | 10–30 V DC |

Every config assumes the sensor's factory defaults: Modbus address 1, 8 data bits, no parity and 1 stop bit, at the baud rate above.

## Hardware

- **Controller:** M5Stack Unit PoE P4 (ESP32-P4 with an IP101 Ethernet PHY), powered over PoE.
- **RS485:** a TTL-to-RS485 transceiver on GPIO53 (TX) and GPIO54 (RX). No flow-control (DE/RE) pin is configured, so use a transceiver with automatic direction control.
- **Sensor power:** check each sensor's supply range in the table above. Most need 10 V or more, so they won't run off the 3.3 V or 5 V an RS485 adaptor passes through; the CWT-SWS-C anemometer simply never answers on 5 V. Use a 12 V or 24 V supply (a PoE-to-12 V splitter works) and connect its negative to the adaptor's GND, since the sensors have no separate signal ground.
- **One sensor per board.** Each config sets up its own UART and Modbus bus and expects its sensor at address 1. To share one bus between several sensors, give each a unique Modbus address and merge them under a single `uart:` / `modbus:` block.

Wire colours vary between CWT models (RS485 A+ is green on some, yellow or blue on others). The header comment in each CWT-* config lists that sensor's wiring; for the THC-S, follow its manual.

## Using the configs

**THC-S substrate probe:** a complete device config, built on the [poep4-preprov](https://github.com/Chill-Division/poep4-preprov) base. Edit the `substitutions:` at the top (device name, Modbus address, poll interval and EC temperature coefficient), then compile and flash.

**Everything else:** sensor-only partials, with just the `uart:`, `modbus:`, `modbus_controller:` and entity blocks and no board or network setup. Combine one with a PoE P4 base config such as [`poep4-preprov.yaml`](https://github.com/Chill-Division/poep4-preprov/blob/main/poep4-preprov.yaml), either by pasting the blocks in or by saving the file next to your device config and including it as a package:

```yaml
packages:
  water_level: !include "esp32 poe-p4 comwintop cwt-wls water level sensor.yaml"
```

All six configs pass `esphome config` on ESPHome 2026.9.0 (the partials when included as above). Older releases may reject the ESP32-P4 board or the THC-S config's `input` register type. Passing validation only means the YAML is well-formed; see the warning at the top.

## Things worth knowing

- **Function codes in the manuals.** Several CWT register tables list function codes "0x30" and "0x60", but the example frames in the same manuals use 0x03 (read holding registers) and 0x06 (write single register). The configs follow the example frames.
- **"Humidity" isn't air humidity.** On the THC-S it's the substrate's volumetric water content (VWC). On the CWT-LEAF-TH it's leaf surface wetness, and a dry pad reads about 0%. The entities are named to match.
- **Estimated pore-water EC is an approximation.** It's bulk EC ÷ VWC rather than the Hilhorst model, which needs a dielectric permittivity reading the THC-S doesn't provide. Use it for trends, not absolute values. It reports unknown below 5% VWC, and it equals bulk EC whenever the probe reads 100% VWC (for example, sitting in a beaker of solution).
- **EC units.** The THC-S reports substrate EC in mS/cm, plus a dS/m copy for agronomy-style dashboards (1 dS/m = 1 mS/cm). The water EC transmitter reports µS/cm, plus an mS/cm copy so reservoir and substrate EC can be compared directly.
- **Calibration.** The leaf sensor (registers 0x0050 and 0x0051) and PAR sensor (0x0052) store calibration offsets in the sensor itself, and the configs expose them as Home Assistant number entities. The water EC transmitter is calibrated with its own button (zero, then a 1413 µS/cm standard); its config only reports the stored calibration value as a diagnostic.
- **EC temperature compensation (THC-S).** The probe ships with it switched off (0.0 %/°C). At boot the config waits for the probe to respond, writes `ec_temp_coeff` (default 2.0 %/°C) only if the stored value differs, then reads it back to confirm, retrying up to three times. Look for `EC temperature coefficient confirmed` in the boot log; this has been confirmed working on our test unit.

## Vendor manuals

The ComWinTop manuals mentioned in the config headers aren't included in this repo, as they're ComWinTop's documents. Get them from ComWinTop or your supplier.

## License

[MIT](LICENSE)
