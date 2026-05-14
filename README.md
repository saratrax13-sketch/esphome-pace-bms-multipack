# ESPHome PACE BMS Multipack Monitor

ESPHome PACE BMS Multipack Monitor is an ESP32-based RS485/Modbus configuration for monitoring multiple PACE-based lithium battery packs in Home Assistant.

The project is designed for battery systems that use the PACE BMS RS485/Modbus register map, including many Hubble AM2-style and other PACE-based lithium battery packs.

It supports per-pack monitoring, automatic valid cell-count detection, combined battery-bank calculations, projected runtime, projected charging time, MQTT/Home Assistant integration, and clear operational fault visibility.

---

## Key Features

- ESP32 + RS485 monitoring
- PACE RS485/Modbus support
- Multi-pack design
- Pack 1 / Master and Pack 2 / Slave 1 active in the base v2 file
- Full Pack 3 / Slave 2 and Pack 4 / Slave 3 code blocks included but commented out by default
- Users can uncomment Pack 3 / Pack 4 sections when extra physical batteries are added
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

## Main YAML File

Recommended YAML file name:

```text
pace-bms-multipack-v2-with-commented-extra-packs.yaml
```

This file contains:

```text
Pack 1 / Master active code
Pack 2 / Slave 1 active code
Pack 3 / Slave 2 full code, commented out
Pack 4 / Slave 3 full code, commented out
Combined calculation guidance
Runtime and charging-time sensors in minutes
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
Pack 1 / Master   = 0x01  active by default
Pack 2 / Slave 1  = 0x02  active by default
Pack 3 / Slave 2  = 0x03  included but commented out
Pack 4 / Slave 3  = 0x04  included but commented out
```

The Pack 3 and Pack 4 code blocks are already included in the YAML file, but they are commented out with `#` so they do not run until the user intentionally enables them.

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

  # Pack 3 / Slave 2 - included in the full YAML but commented out
  # - id: bms_pack_3
  #   address: 0x03
  #   modbus_id: modbus0
  #   setup_priority: -10
  #   command_throttle: 300ms
  #   update_interval: 35s
  #   offline_skip_updates: 3
  #
  # Pack 4 / Slave 3 - included in the full YAML but commented out
  # - id: bms_pack_4
  #   address: 0x04
  #   modbus_id: modbus0
  #   setup_priority: -10
  #   command_throttle: 400ms
  #   update_interval: 45s
  #   offline_skip_updates: 3
```

> Important: The YAML can only monitor packs that are explicitly defined. ESPHome cannot dynamically create new sensors at runtime.

---

## Enabling Pack 3 and Pack 4

The v2 YAML includes full Pack 3 and Pack 4 code blocks, but they are commented out by default.

This makes the file easier to expand later without needing to rewrite the configuration.

Before enabling an extra pack, confirm:

```text
The physical battery is installed
The battery is connected to the same RS485 daisy-chain bus
The battery has a unique Modbus address
The address matches the YAML section being enabled
Only one ESP32 is polling the RS485 bus
```

Default expansion convention:

```text
Pack 3 / Slave 2 = Modbus address 0x03
Pack 4 / Slave 3 = Modbus address 0x04
```

To enable Pack 3:

```text
1. Search the YAML for "OPTIONAL PACK 3".
2. Remove the leading "# " from the Pack 3 Modbus controller block.
3. Remove the leading "# " from the Pack 3 binary sensor block.
4. Remove the leading "# " from the Pack 3 numeric sensor block.
5. Remove the leading "# " from the Pack 3 projected time sensor block.
6. Remove the leading "# " from the Pack 3 text sensor / decoder block.
7. Update the combined calculation lambdas to include Pack 3.
```

To enable Pack 4:

```text
1. Search the YAML for "OPTIONAL PACK 4".
2. Remove the leading "# " from the Pack 4 Modbus controller block.
3. Remove the leading "# " from the Pack 4 binary sensor block.
4. Remove the leading "# " from the Pack 4 numeric sensor block.
5. Remove the leading "# " from the Pack 4 projected time sensor block.
6. Remove the leading "# " from the Pack 4 text sensor / decoder block.
7. Update the combined calculation lambdas to include Pack 4.
```

> Important: Uncommenting the Pack 3 or Pack 4 sensor blocks alone is not enough. The combined battery-bank calculations must also be updated so that totals, averages, minimums, maximums, runtime, and charging-time estimates include the newly enabled packs.


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

---

## RJ45 RS485 Pinout Documentation

Many PACE-based lithium batteries expose RS485 communication through an RJ45-style port. This project can work with RJ45 battery communication ports, but the exact pinout must always be confirmed against the battery manufacturer's documentation.

> Important: RJ45 does not automatically mean Ethernet. On battery BMS systems, an RJ45 socket is often only used as a convenient connector for RS485, CAN, dry contact, or other low-voltage communication signals.

---

### Common RJ45 Numbering

When looking at an RJ45 plug with the clip facing away from you and the copper contacts facing up, pin 1 is usually on the left:

```text
Copper contacts facing up
Clip facing away

Pin:  1  2  3  4  5  6  7  8
      |  |  |  |  |  |  |  |
     [------------------------]
```

Another way to view it:

```text
RJ45 plug front view, contacts up:

  1  2  3  4  5  6  7  8
┌────────────────────────────┐
│ ▓  ▓  ▓  ▓  ▓  ▓  ▓  ▓     │
└────────────────────────────┘
```

Always confirm orientation before crimping or probing.

---

### Common PACE / Pylon-Style RS485 RJ45 Pinout

A commonly used RS485 pinout on many PACE/Pylon-style lithium batteries is:

| RJ45 Pin | Signal | Notes |
|---|---|---|
| Pin 1 | RS485-B / 485B / D- | Inverting RS485 line |
| Pin 2 | RS485-A / 485A / D+ | Non-inverting RS485 line |
| Pin 3 | NC / Reserved | May vary by battery |
| Pin 4 | CAN-H | Used for CAN, not RS485 |
| Pin 5 | CAN-L | Used for CAN, not RS485 |
| Pin 6 | NC / Reserved | May vary by battery |
| Pin 7 | GND / COM | Communication ground on some models |
| Pin 8 | GND / COM | Communication ground on some models |

> Warning: This is a common pattern, not a universal rule. Some batteries swap A/B, use different ground pins, or use separate RJ45 ports for inverter CAN and RS485 monitoring.

---

### RS485 Connection from RJ45 to RS485 Board

If your battery uses the common pinout above, the connection to the RS485 board would normally be:

| Battery RJ45 Pin | Battery Signal | RS485 Board |
|---|---|---|
| Pin 1 | RS485-B / D- | B / D- / 485- |
| Pin 2 | RS485-A / D+ | A / D+ / 485+ |
| Pin 7 or 8 | GND / COM | GND, if required |

Example:

```text
Battery RJ45 Pin 2  RS485-A / D+  ───── RS485 board A / D+ / 485+
Battery RJ45 Pin 1  RS485-B / D-  ───── RS485 board B / D- / 485-
Battery RJ45 Pin 7  GND / COM     ───── RS485 board GND, if required
```

If there is no communication but everything else is correct, try swapping A and B:

```text
Battery A / D+  -> RS485 board B / D-
Battery B / D-  -> RS485 board A / D+
```

Some manufacturers label A/B opposite to the RS485 board manufacturer.

---

### RJ45 Daisy-Chain Between Multiple Batteries

When using multiple batteries, the RS485 bus should continue from one battery to the next.

If the batteries have dual RJ45 link ports, the physical chain may look like this:

```text
ESP32 RS485 board
   |
   | RS485 A/B/GND
   |
RJ45 RS485 port on Pack 1 / Master
   |
   | Battery link cable
   |
RJ45 RS485 port on Pack 2 / Slave 1
   |
   | Battery link cable
   |
RJ45 RS485 port on Pack 3 / Slave 2
   |
   | Battery link cable
   |
RJ45 RS485 port on Pack 4 / Slave 3
```

Electrically, the RS485 bus is still:

```text
A / D+  ───── Pack 1 A ───── Pack 2 A ───── Pack 3 A ───── Pack 4 A
B / D-  ───── Pack 1 B ───── Pack 2 B ───── Pack 3 B ───── Pack 4 B
GND     ───── Pack 1 GND ─── Pack 2 GND ─── Pack 3 GND ─── Pack 4 GND
```

---

### RJ45 Breakout Cable Example

A practical way to connect an ESP32 RS485 board to the battery is to use a short RJ45 breakout cable.

Example using the common PACE/Pylon-style RS485 pinout:

```text
RJ45 Pin 1  -> RS485 B / D-
RJ45 Pin 2  -> RS485 A / D+
RJ45 Pin 7  -> GND / COM, optional if required
RJ45 Pin 8  -> GND / COM, optional if required
```

Then connect to the RS485 board:

```text
RJ45 Pin 1  -> RS485 board B / D-
RJ45 Pin 2  -> RS485 board A / D+
RJ45 Pin 7/8 -> RS485 board GND
```

Do not connect CAN-H or CAN-L to the RS485 board.

```text
RJ45 Pin 4 CAN-H  -> do not connect to RS485 board
RJ45 Pin 5 CAN-L  -> do not connect to RS485 board
```

---

### T568B Wire Colours

If the RJ45 cable uses standard T568B wiring, the wire colours are usually:

| RJ45 Pin | T568B Colour |
|---|---|
| Pin 1 | White/Orange |
| Pin 2 | Orange |
| Pin 3 | White/Green |
| Pin 4 | Blue |
| Pin 5 | White/Blue |
| Pin 6 | Green |
| Pin 7 | White/Brown |
| Pin 8 | Brown |

Using the common PACE/Pylon-style RS485 pinout:

| Function | RJ45 Pin | T568B Colour |
|---|---|---|
| RS485-B / D- | Pin 1 | White/Orange |
| RS485-A / D+ | Pin 2 | Orange |
| GND / COM | Pin 7 or 8 | White/Brown or Brown |

> Always test continuity with a multimeter. Do not rely only on cable colour, especially with pre-made or non-standard cables.

---

### CAN Port vs RS485 Port

Many batteries have multiple RJ45 ports, for example:

```text
CAN / Inverter communication port
RS485 / BMS monitoring port
Link / battery-to-battery port
```

This ESPHome project uses RS485/Modbus.

Do not connect the ESP32 RS485 board to the CAN pins or a CAN-only port.

If the battery has a dedicated RS485 port, use that port.

If the battery uses the same physical RJ45 connector for multiple communication types, confirm which pins are RS485 A/B before connecting.

---

### RJ45 Safety Checklist

Before powering the ESP32/RS485 monitor:

```text
Confirm the battery RJ45 pinout from the manual
Confirm whether the port is RS485, not CAN-only
Confirm RJ45 pin 1 orientation
Confirm A/B wiring with a continuity tester
Confirm GND/COM is connected only if required
Confirm no battery power pins are connected to the RS485 board
Confirm only one ESP32/device is polling the RS485 bus
```

Incorrect RJ45 wiring can prevent communication and may damage hardware if voltage pins are connected incorrectly.


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

### Updating Combined Calculations for Extra Packs

The base v2 file actively calculates combined values for:

```text
Pack 1 + Pack 2
```

If Pack 3 and/or Pack 4 are enabled, the combined calculation lambdas must be updated.

Example current calculation for two packs:

```cpp
return id(pack_1_current).state + id(pack_2_current).state;
```

Example for three packs:

```cpp
return id(pack_1_current).state + id(pack_2_current).state + id(pack_3_current).state;
```

Example for four packs:

```cpp
return id(pack_1_current).state + id(pack_2_current).state + id(pack_3_current).state + id(pack_4_current).state;
```

For voltage, do not add pack voltages. Average valid pack voltages instead.

For capacity, current, and power, add all enabled packs.

For cell health, use:

```text
Lowest cell voltage = lowest valid cell across all enabled packs
Highest cell voltage = highest valid cell across all enabled packs
Worst delta = highest pack delta across all enabled packs
Detected cells = sum of detected valid cells from all enabled packs
```

For data trust, keep this rule:

```text
If an enabled pack is unavailable, the combined value should become unavailable instead of showing a partial total.
```


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


---

## Home Assistant `configuration.yaml` Templates

No Home Assistant `configuration.yaml` template sensors are required for the PACE BMS Multipack project.

The ESPHome YAML creates the required entities directly, including:

```text
Per-pack voltage, current, power, SOC and SOH
Per-pack remaining, full and design capacity
Per-pack cell voltages and balancing states
Per-pack warnings, protections, faults and status
Combined battery-bank totals
Combined cell-health values
Projected runtime
Projected charging time
```

Users may create their own optional Home Assistant templates if they want custom dashboard summaries, but they are not required for the standard setup.

> Important: Do not copy unrelated inverter templates, such as Sunsynk PV/load/grid templates, into this project documentation. They are separate from the PACE BMS Multipack monitor and may confuse users.


## Troubleshooting

### Pack 1 reads, but Pack 2 / Pack 3 / Pack 4 does not

Check:

```text
The affected pack is connected to the same RS485 bus
The affected pack has a unique Modbus address
The affected pack address matches the YAML address
The affected pack code block is uncommented if it is Pack 3 or Pack 4
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
