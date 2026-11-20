# ESPHome BLE bridge for Vevor / BYD-branded diesel heaters

Control a BLE-enabled Vevor diesel air heater (tested on a **Vevor XMZ-W-D3**,
advertising as `BYD-xxxxxxxxxxxx`) from Home Assistant, over WiFi, using a
cheap ESP32 as a Bluetooth-to-WiFi bridge. No app, no cloud, no direct wiring
into the heater's control cable.

## Why this exists

Several open-source projects already document diesel heater BLE protocols
(ABBA/HeaterCC, plain AA55, AA66, CBFF/Sunster, Hcalory, ...), but none of
them matched this specific heater:

- It advertises as `BYD-xxxxxxxxxxxx` and exposes GATT service `0xFFE0` with
  characteristic `0xFFE1` (Read/Write/Notify) — the same UUIDs used by
  several "plain AA55" Vevor projects.
- Writing the plain/unencrypted AA55 command format those projects use
  produces no response at all.
- The heater *does* respond — but the 8-byte **command** frame it expects
  has an extra `0x0C 0x22` marker not seen elsewhere, and its **status
  notification** is 48 bytes long and XOR-"encrypted" with the ASCII key
  `"password"`.

Both of these were reverse-engineered by comparing live BLE captures
(nRF Connect) against the (unencrypted) AA66 frame layout documented in the
[`diesel-heater-ble`](https://pypi.org/project/diesel-heater-ble/) Python
library, and against
[espnomad/ESPHome-BLE-Dieselheater](https://github.com/espnomad/ESPHome-BLE-Dieselheater)
(same UUIDs, different, shorter, unencrypted command frame). Neither
documents the `"password"` XOR key or the `0x0C 0x22` command marker, so
this is published in case it saves someone else the same debugging loop.

## Hardware

- Any ESP32 dev board (this was built and tested on a plain `esp32dev`)
- A solid 5V power supply for the ESP32 — a flaky one causes random reboots
  during BLE radio activity that look like firmware bugs but aren't.

## Protocol summary

GATT service: `0xFFE0`
GATT characteristic: `0xFFE1` (Read, Write, Notify)

### Command frame (write to 0xFFE1, 8 bytes)

```
AA 55 0C 22 <command> <val1> <val2> <checksum>
checksum = (0xAA + 0x55 + 0x0C + 0x22 + command + val1 + val2 + 1) mod 256
```

| command | meaning                | val1                                  |
|--------:|------------------------|----------------------------------------|
| 1       | query status           | 0                                       |
| 2       | set mode               | 1 = level mode, 2 = temperature/auto mode |
| 3       | power                  | 1 = on, 0 = off                         |
| 4       | set level/temperature  | the value; applies to whichever mode was last selected with command 2 |

Send command 1 at least every few seconds to keep status notifications
flowing — the heater does not appear to push status on its own schedule
alone.

### Status notification (push notify from 0xFFE1, 48 bytes on the wire)

The 48 bytes are XOR-"encrypted" with the repeating 8-byte key `"password"`
(ASCII bytes `70 61 73 73 77 6F 72 64`), applied block-wise:

```
decrypted[i] = raw[i] XOR key[i % 8]      for i in 0..47
```

The decrypted frame starts with `AA 66 01 ...`. Relevant byte offsets
(0-indexed, after decryption):

| offset   | field                                    |
|---------:|-------------------------------------------|
| 3        | running_state (0 = off, >0 = on)           |
| 5        | running_step (0/1/2 = starting, 3 = heating, 4 = cooldown, 6 = fan only) |
| 6–7      | altitude, big-endian, ÷10                  |
| 8        | running_mode (1 = level, 2 = temperature)  |
| 9        | set_temp (°C)                              |
| 10       | set_level (1–10)                           |
| 11–12    | supply_voltage, big-endian, ÷10            |
| 13–14    | case_temperature, big-endian, signed 16-bit |
| 32–33    | cab_temperature, big-endian, signed 16-bit, ÷10 |
| 35       | raw error byte; `error_code = raw & 0x3F`  |

This matches the *unencrypted* AA66 layout documented in `diesel-heater-ble`
almost field-for-field once decrypted — the only differences are the frame
length (48 vs. 20 bytes) and a few extra configuration bytes (language, tank
volume, pump/RF433 flag, backlight) carried in the unused space, which
aren't exposed by this YAML but are easy to add if you need them.

## Setup

1. Find your heater's MAC address (e.g. with the nRF Connect app — look
   for a device named `BYD-xxxxxxxxxxxx`).
2. Copy `secrets.yaml.example` to `secrets.yaml` and fill in your WiFi
   credentials.
3. Edit `vevor-byd-ble-heater.yaml`: set `heater_mac` under `substitutions`
   to your heater's MAC address.
4. Flash with ESPHome (`esphome run vevor-byd-ble-heater.yaml`) or via the
   ESPHome dashboard/add-on.
5. The device should auto-discover in Home Assistant via mDNS. You'll get:
   - a **Heater Power** switch
   - a **Heater Power Level** select (1–10)
   - sensors for voltage, case/cabin temperature, error code
   - a status text sensor and a raw debug hex sensor (useful if you need to
     adapt this for a different heater variant)

## Caveats / things we haven't nailed down

- The exact meaning of a "temperature mode" set-point (`set_temp` at byte 9)
  hasn't been exercised — this heater was tested in level mode.
- Error code strings aren't mapped yet (the raw code is exposed, but not
  translated to readable text).
- Not tested against the AA55 (non-BYD) variant — if your heater doesn't
  advertise as `BYD-...`, this probably won't apply as-is; check the
  `diesel-heater-ble` library's protocol table first.

## Credits

- [`diesel-heater-ble`](https://pypi.org/project/diesel-heater-ble/) (PyPI) —
  protocol reference for the AA55/AA66 byte layout and command structure.
- [espnomad/ESPHome-BLE-Dieselheater](https://github.com/espnomad/ESPHome-BLE-Dieselheater) —
  confirmed the `0xFFE0`/`0xFFE1` UUIDs and a related (unencrypted) command
  frame shape for Vevor-branded heaters.
- [mSoftMS/AirHeater-BLE](https://github.com/mSoftMS/AirHeater-BLE) —
  protocol documentation for the related Hcalory/AirHeater BLE variant
  (different UUIDs: `0xFFF0`/`0xFFF1`/`0xFFF2`).

## License

MIT — do whatever you want with this, no warranty. This controls a
fuel-burning heater; test carefully and keep an eye on it.
