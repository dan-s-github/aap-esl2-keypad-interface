# AAP ESL-2 Keypad Interface

ESPHome firmware and a custom PCB to talk to the keypad bus of an [Arrowhead Alarm Products](https://www.aap.co.nz) EliteControl **ESL-2** panel, so it can be integrated into Home Assistant through the [`crow_alarm_panel`](https://github.com/dan-s-github/esphome-components) external component. It builds on my `crow_alarm_panel` implementation, which handles the AAP keypad bus protocol.

| PCB | PCB with Atom Lite | Side View |
| --- | --- | --- |
| ![Custom PCB][custom_pcb] | ![Custom PCB with Atom Lite][custom_pcb_atom] | ![Custom PCB on the side][custom_pcb_side] |

The ESL-2 keypad bus is a shared clock/data bus that regular AAP keypads talk to. This project adds an [M5Stack Atom Lite](https://docs.m5stack.com/en/core/Atom%20Lite) or [M5Stack AtomS3 Lite](https://docs.m5stack.com/en/core/AtomS3%20Lite) to the bus as an extra keypad, so it can read zone/arm state and issue arm/disarm/output commands over Wi-Fi, without touching the panel's existing keypads.

  ![Installed][installed]

## How it works

The panel supplies +12V to power the bus, but the clock/data lines themselves run at 5V, while the AtomS3 Lite module's GPIOs are 3.3V. The custom PCB sits between the two:

- An [`AP63205WU-7`](https://www.diodes.com/part/view/AP63205) buck converter steps the bus's +12V supply down to +5V, used as the CLK/DATA logic-high reference and to power the module.
- Two `BSS138` MOSFETs form bidirectional logic-level shifters for the CLK and DATA lines, translating between the bus side (5V) and the module side (3.3V).
- The PCB's pin header/Grove-compatible footprint matches the Atom Lite, so it plugs straight onto the PCB with no jumper wires — only the bus connection (panel to PCB) needs wiring, covered in [Connection diagram](#connection-diagram).

The header footprint matches the M5Stack ATOM family in general, so the original [M5Stack Atom Lite](https://docs.m5stack.com/en/core/ATOM%20Lite) (ESP32) works as a drop-in alternative to the AtomS3 Lite — it needs the plain `esp32` variant instead of `esp32s3`, plus different `clock_pin`/`data_pin` values in the YAML, since the Atom Lite exposes a different set of GPIOs than the AtomS3 Lite.

There is no dedicated address pin/DIP switch — the keypad address used on the bus is set entirely in the ESPHome YAML (see [Configuration](#configuration) below).

## Custom PCB

All manufacturing files (schematic, gerbers, BOM, pick-and-place, and the EasyEDA source project) are in [`docs/pcb`](docs/pcb/), ready to order from a board house such as [JLCPCB](https://jlcpcb.com/).

  ![Schematics](docs/pcb/SCH_AAP%20Bus%20Interface_1-Schematics_2026-07-08.png)

- [Schematics PDF](docs/pcb/SCH_AAP%20Bus%20Interface_2026-07-08.pdf)
- [EasyEDA project](docs/pcb/ProPrj_AAP%20Bus%20Interface%20v2_2026-07-08.epro2) / [EasyEDA doc](docs/pcb/ProDoc_AAP%20Bus%20Interface_2026-07-08.epro2)

Files required to order boards from JLCPCB:

- [Gerber files](docs/pcb/Gerber_PCB_AAP_Bus_Interface_2026-07-08.zip)
- [Bill of Materials](docs/pcb/BOM_AAP%20Bus%20Interface_PCB_AAP%20Bus%20Interface_2026-07-08.csv)
- [Pick and Place](docs/pcb/PickAndPlace_PCB_AAP%20Bus%20Interface_2026_07_08.csv)

### Board layout

The top side mounts the AtomS3 Lite and carries the buck converter and level shifters. The bottom side breaks out the bus connection, silkscreened with the wire colors used on the ESL-2 keypad harness:

| Silkscreen | Signal      | Wire color |
|------------|-------------|------------|
| +12V       | Bus power   | Red        |
| GND        | Ground      | Black      |
| CLK        | Clock       | Yellow     |
| DAT        | Data        | Blue       |

| Top | Bottom |
| --- | ------ |
| ![2D Top][custom_pcb_2D_top] | ![2D Bottom][custom_pcb_2D_bottom] |
| ![3D Top][custom_pcb_3D_top_1] | ![3D Bottom][custom_pcb_3D_bottom] |
| ![3D Top][custom_pcb_3D_top_2] | |

Two options are provided for connecting to the bus:

- `CN1` — a 4-pin screw terminal (Phoenix Contact 1861959), for connecting bare wires directly.
- `CN2` — a 5-pin pluggable connector (Molex 22035055), for a crimped/pre-made harness — compatible with AAP's Keypad Bus Connection Cables. [ARR14](https://www.aap.co.nz/shop/Alarm+Systems/Keypads/ARR14.html), [ARR15](https://www.aap.co.nz/shop/Alarm+Systems/Keypads/ARR15.html)

### Connection diagram

```txt
+--------------------+                +-----------------+
|    ESL-2 Panel     |                |    Custom PCB   |
|                    |                |                 |
| +12V (red)    o---------------------o +12V            |
| GND  (black)  o---------------------o GND             |
| CLK  (yellow) o---------------------o CLK             |
| DAT  (blue)   o---------------------o DAT             |
+--------------------+                +-----------------+
```

## Configuration

Two example YAML configs are provided, both using the same PCB and wiring. Copy the one you need (dropping `.example`) and fill in your own zone/keypad names — the copy is gitignored so your real panel layout doesn't end up in the repo:

- [`aap_esl_keypad_interface.example.yaml`](aap_esl_keypad_interface.example.yaml) → `aap_esl_keypad_interface.yaml` — registers as an active keypad on the bus (`address: 5`) and exposes arm/disarm/stay, zone sensors, and switch outputs to Home Assistant.
- [`aap_esl_keypad_monitor.example.yaml`](aap_esl_keypad_monitor.example.yaml) → `aap_esl_keypad_monitor.yaml` — passive/listen-only variant (no `address` claimed on the bus), useful for monitoring bus traffic without presenting as a keypad.

Both use `clock_pin: GPIO7`, `data_pin: GPIO5` for the AtomS3 Lite (`GPIO23`/`GPIO22` instead if using an original Atom Lite, along with `esp32` instead of `esp32s3`) with the [`crow_alarm_panel`](https://github.com/dan-s-github/esphome-components) external component. For everything else — `keypads`, `zones`, passive monitor mode, outputs, bypass switches, the full option reference — see [the component's own README](https://github.com/dan-s-github/esphome-components/blob/main/components/crow_alarm_panel/README.md), which documents the schema in more detail than duplicated here.

Create a `secrets.yaml` (gitignored) alongside your copy with:

```yaml
wifi_ssid: "..."
wifi_password: "..."
fallback_hotspot_password: "..."
alarm_code: 1234
api_encryption_key: "..."
```

### Flash

Two ways to get this onto the device:

**Local CLI** — this project uses [`uv`](https://docs.astral.sh/uv/) to manage the ESPHome dependency (see [`pyproject.toml`](pyproject.toml)). [Install `uv`](https://docs.astral.sh/uv/getting-started/installation/) if you don't have it, then:

```sh
uv sync
uv run esphome config aap_esl_keypad_interface.yaml  # to validate configuration
uv run esphome run aap_esl_keypad_interface.yaml
```

**ESPHome Builder (Home Assistant add-on)** — no local ESPHome install needed:

1. In Home Assistant, open the ESPHome dashboard and `+ NEW DEVICE` to create a device and flash the stub firmware over USB (this sets up `wifi`, `api`, and `ota` for you, with the encryption key stored by the add-on).
2. Once the device shows up and adopts, `EDIT` it and replace the generated YAML's `esp32`, `logger`, `external_components`, and `crow_alarm_panel`/`text_sensor`/`alarm_control_panel`/`button`/`switch` blocks with the equivalent sections from [`aap_esl_keypad_interface.example.yaml`](aap_esl_keypad_interface.example.yaml), filled in with your own zones/keypads.
3. Leave the wizard-generated `wifi`/`api`/`ota` sections as-is — no need to bring in `secrets.yaml` since the add-on manages Wi-Fi credentials and the encryption key itself.
4. `INSTALL` to push the update over-the-air.

## Credits

[`crow_alarm_panel`](https://github.com/dan-s-github/esphome-components) was built on the work done by [jallier](https://github.com/jallier/esphome-components/tree/main/components) and [jesserockz](https://github.com/jesserockz/esphome-components/tree/main/components/crow_alarm_panel)

[custom_pcb]: docs/custom_pcb.png
[custom_pcb_atom]: docs/custom_pcb_atom.png
[custom_pcb_side]: docs/custom_pcb_side.png

[custom_pcb_2D_top]: docs/custom_pcb_2D_top.png
[custom_pcb_2D_bottom]: docs/custom_pcb_2D_bottom.png

[custom_pcb_3D_top_1]: docs/custom_pcb_3D_top_1.png
[custom_pcb_3D_top_2]: docs/custom_pcb_3D_top_2.png
[custom_pcb_3D_bottom]: docs/custom_pcb_3D_bottom.png

[installed]: docs/installed.png
