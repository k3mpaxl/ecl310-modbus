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
register map.

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
