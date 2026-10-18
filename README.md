# ecl310-modbus

A device library for reading and writing a **Danfoss ECL Comfort 310** heating
and district heating controller over Modbus TCP or a Modbus RTU gateway. It
talks to the controller through
[`modbus-connection`](https://github.com/home-assistant-libs/modbus-connection),
so it works with either `pymodbus` or `tmodbus` as the underlying client.

This is the device library behind
[`ecl310-hacs`](https://github.com/k3mpaxl/ecl310-hacs), the Home Assistant
custom integration. It carries no Home Assistant dependency itself - just the
register map, decoding and validation for the controller.

## Usage

```python
from modbus_connection import ModbusUnit
from ecl310_modbus import Ecl310

unit = ModbusUnit(...)  # see modbus-connection for connection setup
device = Ecl310(unit)
await device.async_update()

device.sensors.outdoor_temperature
device.heating.comfort_setpoint
device.hot_water.setpoint
```

Sub-systems live in `ecl310_modbus.subsystems`. Address helpers, the neutral
datapoint metadata and the static configuration tables live in
`ecl310_modbus.addresses`, `ecl310_modbus.metadata` and
`ecl310_modbus.configurations`.

Every field a component declares carries a `DatapointMetadata` next to its
address: the parameter ID the ECL manual prints, the value domain the
controller accepts, the unit, and whether the value may be written. Consumers
can use it to build user interfaces without hard-coding a second copy of the
register map. A setting with an "off" value outside its range - the
disinfection temperature stores `9` for `OFF` - names it in `off_value`.

## Applications

An ECL controller runs one *application* (A237.1, A337.2, ...), and the
application decides which parameters it has; a parameter it lacks is refused
over Modbus. `ecl310_modbus.applications` carries a profile per application
and version - every parameter, with type, scale, range, default and whether it
may be written:

```python
from ecl310_modbus import PROFILES, Ecl310

detection = await Ecl310.async_detect_application(unit)
profile = detection.application or PROFILES["a237_1_v04"]
device = Ecl310(unit, application=profile)
device.supports("heating", "return_limit")  # False on A337.2
device.info.application                     # "A337.2 V02"
```

With a profile, the device reads only the parameters the application has.

Detection first asks the controller: it reports its application in PNU
2060-2063 (`detection.reported`, e.g. `"A237.1 V04"`). A controller that reports
an application without a profile here gets `detection.application is None`.
Only a controller that reports nothing is probed: a handful of parameters that
tell the known applications apart are read one by one, and the profiles are
ranked by how many answers each predicts wrongly.

## Identity

The controller reports what it is (Danfoss VI.LG.V5.02, §6.12):

```python
system = await device.async_read_system_information()
system.application_title   # "A237.1 V04"   PNU 2060-2063
system.software            # "1.56"         PNU 35 (build in PNU 8)
system.hardware            # "H.3"          PNU 34
system.code_number         # "087H3040"     PNU 19
system.serial_number       # "66069"        PNU 36-37
system.production          # (2017, 12)     PNU 2099, year and week
device.info                # the same, for a device registry
```

It is read once and not polled, because none of it changes while the controller
runs. `Ecl310.async_probe()` reads it too.

| Profile | Application | Version |
| :------ | :---------- | :------ |
| `a237_1_v04` | A237.1 | V04 |
| `a266_10_v03` | A266.10 | V03 |
| `a337_2_v02` | A337.2 | V02 |

Each parameter carries the following, for consumers that want to label or
group what they show:

* its circuit (`1` heating, `2` hot water, `0` controller-wide)
* the controller display's German text (`parameter.display_name("de")`, e.g.
  "Heizkurve")
* the meaning of its raw values, where the report lists them

A profile also says what the sensor inputs measure (`sensors`) and what the
relays drive (`relays`). The device reads the sensors accordingly, so on
A266.10 `S4` is the hot water flow and `S6` the hot water return.
`device.relay_assignments` is `None` while an application's relays are not
known.

The profiles are generated from the reports in Danfoss' ECL Tool - only the
derived parameter metadata, not the reports themselves:

```sh
scripts/import-ecl-tool "ECL Tool.zip" "A237_1 V04" "A337_2 V02" "A266_10 V03"
```

## Verified against

A **Danfoss ECL Comfort 310**, order number 087H3040, application key
**A237.1 V04**, software 1.56, over Modbus/TCP on unit 1.

## Contributing

Issues and pull requests are welcome - a new datapoint, a decoding fix, or a
register range. Home Assistant entity behavior belongs in
[`ecl310-hacs`](https://github.com/k3mpaxl/ecl310-hacs) instead; this repo is
just the device model.

## License

Apache-2.0. See [LICENSE](LICENSE).
