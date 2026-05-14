# ESPHome PACE BMS Multipack Monitor

ESPHome PACE BMS Multipack Monitor is an ESP32-based RS485/Modbus configuration for monitoring multiple PACE-based lithium battery packs in Home Assistant.

The project is designed for battery systems that use the PACE BMS RS485/Modbus register map, including many Hubble AM2-style and other PACE-based lithium battery packs.

It supports per-pack monitoring, automatic valid cell-count detection, combined battery-bank calculations, projected runtime, projected charging time, MQTT/Home Assistant integration, and clear operational fault visibility.

---

## Key Features

- ESP32 + RS485 monitoring
- PACE RS485/Modbus support
- Multi-pack design
- Pack 1 / Master and Pack 2 / Slave 1 support in the base v2 file
- Clear comments for future Pack 3 / Slave 2 and Pack 4 / Slave 3 expansion
- Automatic valid cell-count detection
- Supports 13S and 16S-style PACE BMS layouts
- Per-pack cell voltage monitoring
- Per-pack balancing status
- Per-pack warning, protection, fault, and status decoding
- Combined battery-bank calculations
- Projected runtime while discharging
- Projected charging time while charging
- MQTT discovery support for Home Assistant
- Timeout handling to avoid stale values being shown as live data

---

## Repository Name

Suggested repository name:

```text
esphome-pace-bms-multipack
```

Suggested GitHub description:

```text
ESPHome monitor for multi-pack PACE RS485/Modbus lithium batteries with auto cell-count detection and Home Assistant/MQTT integration.
```

---

## Hardware Required

- ESP32 development board
- RS485-to-TTL board/module
- PACE-based lithium battery/BMS
- Suitable RS485 cable
- Optional: twisted pair cable for RS485 A/B
- Optional: common ground wire if required by your RS485 board/battery setup

---

## ESP32 to RS485 Board Wiring

The ESP32 connects to the RS485 board using UART TX/RX and a flow-control pin.

Example wiring used in this project:

| ESP32 Pin | RS485 Board Pin | Purpose |
|---|---|---|
| GPIO16 | RX / RO | ESP32 receives data from RS485 board |
| GPIO17 | TX / DI | ESP32 sends data to RS485 board |
| GPIO18 | DE / RE | RS485 transmit/receive direction control |
| GND | GND | Common ground |
| 3.3V or 5V | VCC | Power for RS485 board, depending on board requirements |

> Important: Check your RS485 board voltage requirements. Some RS485 modules are 3.3V compatible, while others expect 5V. Do not power a 3.3V-only board from 5V unless it is rated for it.

Typical ESPHome UART configuration:

```yaml
uart:
  - id: uart_0
    tx_pin: GPIO17
    rx_pin: GPIO16
    baud_rate: 9600
    stop_bits: 1
    parity: NONE
```

Typical Modbus configuration:

```yaml
modbus:
  - id: modbus0
    uart_id: uart_0
    flow_control_pin: GPIO18
    send_wait_time: 500ms
```

---

## RS485 Board to Battery Wiring

The RS485 side of the RS485 board must connect to the RS485 communication terminals or pins on the battery/BMS.

Typical RS485 signal naming:

| RS485 Board | Battery/BMS |
|---|---|
| A / D+ / 485+ | A / D+ / 485+ |
| B / D- / 485- | B / D- / 485- |
| GND / COM | GND / COM, if required |

Different manufacturers sometimes label A/B differently. If the BMS does not communicate, and all other settings are correct, try swapping A and B.

---

## Multi-Pack RS485 Bus Wiring

For one ESP32 to read multiple battery packs, all packs must be connected on the same RS485 bus.

Use a daisy-chain / bus layout:

```text
ESP32 RS485 Board
   A / D+  ───── Pack 1 A / D+  ───── Pack 2 A / D+  ───── Pack 3 A / D+  ───── Pack 4 A / D+
   B / D-  ───── Pack 1 B / D-  ───── Pack 2 B / D-  ───── Pack 3 B / D-  ───── Pack 4 B / D-
   GND     ───── Pack 1 GND     ───── Pack 2 GND     ───── Pack 3 GND     ───── Pack 4 GND
```

Recommended physical layout:

```text
┌──────────────┐       ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ ESP32 RS485  │       │ Pack 1       │       │ Pack 2       │       │ Pack 3       │
│ Board        │       │ Master       │       │ Slave 1      │       │ Slave 2      │
│              │       │              │       │              │       │              │
│ A / D+  ─────┼───────┤ A / D+  ─────┼───────┤ A / D+  ─────┼───────┤ A / D+       │
│ B / D-  ─────┼───────┤ B / D-  ─────┼───────┤ B / D-  ─────┼───────┤ B / D-       │
│ GND     ─────┼───────┤ GND     ─────┼───────┤ GND     ─────┼───────┤ GND          │
└──────────────┘       └──────────────┘       └──────────────┘       └──────────────┘
```

Avoid a star layout where all batteries branch separately from the ESP32:

```text
                 ┌──── Pack 1
ESP32 RS485 ─────┼──── Pack 2
                 └──── Pack 3
```

RS485 works best as a bus/daisy-chain.

---

## Modbus Addressing

Each battery pack must have a unique Modbus address.

The base v2 file uses this address map:

```text
Pack 1 / Master   = 0x01
Pack 2 / Slave 1  = 0x02
```

Suggested future expansion convention:

```text
Pack 3 / Slave 2  = 0x03
Pack 4 / Slave 3  = 0x04
```

Example ESPHome Modbus controller section:

```yaml
modbus_controller:
  # Pack 1 / Master
  - id: bms_pack_1
    address: 0x01
    modbus_id: modbus0
    setup_priority: -10
    command_throttle: 500ms
    update_interval: 30s
    offline_skip_updates: 5

  # Pack 2 / Slave 1
  - id: bms_pack_2
    address: 0x02
    modbus_id: modbus0
    setup_priority: -10
    command_throttle: 750ms
    update_interval: 45s
    offline_skip_updates: 5

  # Future expansion:
  #
  # Pack 3 / Slave 2
  # - id: bms_pack_3
  #   address: 0x03
  #   modbus_id: modbus0
  #
  # Pack 4 / Slave 3
  # - id: bms_pack_4
  #   address: 0x04
  #   modbus_id: modbus0
```

> Important: The YAML can only monitor packs that are explicitly defined. ESPHome cannot dynamically create new sensors at runtime.

---

## Important RS485 Notes

Only one RS485 master/poller should be active on the bus.

Do not leave multiple ESP32 devices polling the same battery bus at the same time, for example:

```text
Old ESP32 Master Monitor
Old ESP32 Slave Monitor
New ESP32 Multipack Monitor
```

That can cause collisions, partial responses, timeouts, and unreliable readings.

For normal operation, use one ESP32 multipack monitor connected to the complete RS485 battery bus.

---

## Battery Connection Notes

Many lithium batteries use RJ45 communication ports for CAN/RS485. Do not assume the CAN port and RS485 port are the same.

Check the battery documentation or pinout for:

```text
RS485 A / D+
RS485 B / D-
GND / COM, if required
```

If your battery has multiple communication ports, make sure the ESP32 RS485 board is connected to the correct RS485 BMS port.

---

## Auto Cell-Count Detection

The configuration reads up to 16 possible cell-voltage registers per pack.

Only realistic lithium cell voltages are counted as valid cells.

Typical valid range:

```text
2.0V to 5.0V
```

This allows the same logic to work with packs such as:

```text
13S PACE-based packs
16S PACE-based packs
```

Example behaviour:

```text
13S pack:
  Cell 1-13 = valid voltage
  Cell 14-16 = unavailable / NaN
  Detected cell count = 13

16S pack:
  Cell 1-16 = valid voltage
  Detected cell count = 16
```

The unused cells are not included in:

```text
Minimum cell voltage
Maximum cell voltage
Average cell voltage
Delta cell voltage
Detected cell count
Combined battery calculations
```

---

## Combined Battery Calculations

For batteries connected in parallel, combined values are calculated as follows:

| Combined Value | Calculation |
|---|---|
| Total Current | Add all pack currents |
| Total Power | Add all pack power values |
| Total Remaining Capacity | Add all remaining Ah values |
| Total Full Capacity | Add all full capacity Ah values |
| Total Design Capacity | Add all design capacity Ah values |
| Average Pack Voltage | Average valid pack voltages |
| Combined SOC | Weighted by remaining capacity and full capacity |
| Combined SOH | Based on full capacity versus design capacity |
| Total Detected Cells | Add detected cells from all packs |
| Lowest Cell Voltage | Lowest valid cell across all packs |
| Highest Cell Voltage | Highest valid cell across all packs |
| Worst Delta Cell Voltage | Highest per-pack delta |

> Do not add pack voltages together when packs are connected in parallel.

---

## Projected Runtime and Charging Time

The project includes projected runtime and charging-time sensors.

Runtime while discharging:

```text
Remaining capacity Ah ÷ discharge current A = runtime
```

Charging time while charging:

```text
(Full capacity Ah - remaining capacity Ah) ÷ charge current A = time to full
```

The projected time sensors publish in minutes to avoid Home Assistant displaying sub-1-hour values as `0 h`.

Example:

```text
Projected Runtime: 45 min
Projected Charging Time: 38 min
```

The sensors intentionally return unavailable/NaN when not applicable:

```text
Runtime while charging = unavailable
Charging time while discharging = unavailable
Charging time when battery is already full = 0 min
Runtime when current is near zero = unavailable
```

This avoids misleading values such as extremely high runtime estimates when current is close to zero.

---

## Data Trust and Timeout Behaviour

The configuration is designed not to hide communication failures by holding stale values forever.

The recommended behaviour is:

```text
Good fresh value received = publish value
Invalid/impossible value = reject value
No fresh live value within timeout = unavailable
```

This helps ensure that Home Assistant cards show actual live data rather than old values that make a failed link look healthy.

---

## MQTT and Home Assistant

Suggested MQTT identity for the generic project:

```yaml
mqtt:
  topic_prefix: pace_bms/multipack
```

Suggested ESPHome identity:

```yaml
substitutions:
  name: pace-bms-multipack
  friendly_name: "PACE BMS Multipack"
  device_description: "Multi-pack PACE RS485/Modbus battery monitor with automatic cell-count detection"
```

Example Home Assistant entities:

```text
sensor.pace_bms_multipack_pack_1_state_of_charge
sensor.pace_bms_multipack_pack_2_state_of_charge
sensor.pace_bms_multipack_combined_state_of_charge
sensor.pace_bms_multipack_combined_total_current
sensor.pace_bms_multipack_combined_projected_runtime
sensor.pace_bms_multipack_combined_projected_charging_time
```

---

## Troubleshooting

### Pack 1 reads, but Pack 2 does not

Check:

```text
Pack 2 is connected to the same RS485 bus
Pack 2 has a unique Modbus address
Pack 2 address matches the YAML address
RS485 A/B wiring is correct
Only one ESP32 is polling the bus
```

### All readings are unavailable

Check:

```text
ESP32 UART pins
RS485 board power
RS485 DE/RE flow-control pin
RS485 A/B polarity
Battery RS485 port selection
Modbus baud rate
Battery communication address
```

### Partial responses or Modbus timeouts

Check:

```text
RS485 wiring quality
Cable length
Termination resistor
Multiple devices polling the same bus
Update intervals too aggressive
MQTT overload from too many sensors
```

### Cell 14-16 show unavailable

This is normal for 13S packs.

The sensors remain available in the YAML for compatibility with 16S packs, but they are ignored in calculations when invalid.

### Projected charging time shows 0 min

This is normal when the battery is already full.

Example:

```text
Full capacity = 90.30 Ah
Remaining capacity = 90.30 Ah
Capacity needed = 0 Ah
Charging time = 0 min
```

### Projected runtime shows unavailable

This is normal when the battery is not discharging or when current is too close to zero.

---

## Safety Notes

This project is for monitoring only.

It should not write to the BMS or change battery settings.

Always follow battery manufacturer wiring guidelines, RS485 pinout documentation, and electrical safety practices.

---

## License

Choose a license that suits your intended use.

Suggested open-source license:

```text
MIT License
```

---

## Disclaimer

This project is provided for monitoring and educational purposes. Use it at your own risk. Battery systems can be hazardous. Incorrect wiring or configuration may damage equipment or create unsafe conditions.
