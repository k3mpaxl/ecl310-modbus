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
Detection reads a handful of parameters that tell the known applications apart
and ranks the profiles by how many answers each predicts wrongly.

| Profile | Application | Version |
| :------ | :---------- | :------ |
| `a237_1_v04` | A237.1 | V04 |
| `a337_2_v02` | A337.2 | V02 |

The profiles are generated from the reports in Danfoss' ECL Tool - only the
derived parameter metadata, not the reports themselves:

```sh
scripts/import-ecl-tool "ECL Tool.zip" "A237_1 V04" "A337_2 V02"
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
