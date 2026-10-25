# Generic IOC ioc-plc

PLCs and digital I/O on plain EPICS records over Modbus TCP, for GEECS
(geecs-epics decision 14: setpoint devices are plain records; decision 11:
one image per family). Built on epics-containers' `epics-base` developer
image with the template's iocStats, pvlogging and autosave, plus:

| Module | Version | Source | Reuse (decision 10) |
|---|---|---|---|
| asyn | R4-46 | upstream ibek-support recipe | tier 1, dependency |
| modbus | R3-4 (2024-12) | [epics-modules/modbus](https://github.com/epics-modules/modbus) (APS, Mark Rivers; synApps licence), upstream ibek-support recipe, unchanged | tier 1, dependency |

No epics-containers runtime image carries modbus (ioc-generic,
ioc-streamdevice and the others were checked on 2026-10-24; modbus is in
ibek-support's "general" build group but in no published image), hence this
one. First device: the AutomationDirect Productivity1000 `U_PLC`
(geecs-epics `services/plctest1`).

## Build

No vendor code is inside, so the image could be published; until the
organisation's package registry is set up it is built where it runs:

```
docker build --target runtime -t ioc-plc:local .                              # on the IOC host
DOCKER_DEFAULT_PLATFORM=linux/amd64 TARGET=runtime TAG=ioc-plc:local ./build  # a workstation (amd64, emulated on Apple silicon)
```

The compose files in geecs-epics name `ioc-plc:local`; the pin is this
repository's commit, noted there.

## Entities

Upstream's `modbus` support yaml, nothing of ours:

- `asyn.AsynIP` — the TCP connection (`host:502`; no end-of-string: Modbus
  TCP frames carry their length), then `modbusInterposeConfig(port, 0,
  <timeout ms>, 0)` as a startup command (Modbus TCP framing).
- `modbus.modbusAsynConfigure` — one `drvModbusAsynConfigure` port per
  function code and block (`FUNC_CODE`, `STARTADDR` 0-based, `DATA_LENGTH`,
  `POLL_DELAY` in ms).
- `modbus.modbusPortConfigure` does the first two in one, with the address
  split into `IP` and `TCPPORT`; geecs-epics uses `asyn.AsynIP` so the
  address can come from one environment variable.

Records come from the instance (`dbLoadTemplate` of a substitutions file in
its config directory): upstream's entities cover only 32-bit and float
readouts, and a PLC's channels are named per site.

## What the driver sends at start

Read in the sources before anything ran against a PLC (modbus R3-4
`drvModbusAsyn.cpp`, asyn R4-46 `devAsynUInt32Digital.c`):

1. `drvAsynIPPortConfigure` and `modbusInterposeConfig` connect and send
   nothing.
2. `drvModbusAsynConfigure` for a **read** function (1, 2, 3, 4) starts a
   poller thread that sends that read every `POLL_DELAY` ms, from creation
   on, and posts changed values to `I/O Intr` records once iocInit allows.
3. `drvModbusAsynConfigure` for a **write** function (5, 6, 15, 16) starts
   no poller. If `POLL_DELAY` is non-zero (and addressing is relative,
   `STARTADDR` ≥ 0), the constructor sends **one read** of the same block —
   FC1 for coil writes, FC3 for register writes — and keeps the values
   (`readOnceFunction_`, `readOnceDone_`). With `POLL_DELAY` 0 it sends
   nothing.
4. At iocInit, asyn's `initBo` (and `initAo`, `initMbbo`, `initLo`) calls
   the driver's read: for a write port that returns the read-once values
   with no Modbus traffic (or an error if there was no read-once, and the
   record is left unconverted). An output record therefore starts equal to
   the device and sends nothing. asyn's enum read is skipped (the driver has
   no `asynEnum` interface), and `asyn:READBACK` is not set by our
   templates.
5. A write is sent only when an output record processes: a CA/PVA put, a
   `dbpf`, `PINI`, a scan or a link. In R3-4, modbus's own output templates
   (`bo_bit`, `bo_word`, `ao`, `mbbo`, ...) set no `PINI`, except an
   optional one in `longoutInt32` (default `NO`); `statistics.template`'s
   `MsPerBin` has `PINI 1` but writes a driver parameter, not the device.
   The geecs-epics instances load their own templates.

So an IOC that loads only bit/word output records without `PINI` and has
no autosave of them starts with reads only. geecs-epics proves it per
instance on a Modbus fake that logs any non-read function code as `WRITE`
(`sim/plc_sim.py`) and counts the function codes in the asyn trace
(`services/plctest1/trace_fc.py`).

Notes for configs:

- `DATA_TYPE` is irrelevant for bit access through `asynUInt32Digital`
  (the driver hands back the raw word); 0 (`INT16`) is fine for coils.
- A Productivity tag's Modbus address is assigned in the PLC project
  (Productivity Suite's tag database, "Mod Start"); 32-bit tags take two
  registers, word-swapped by default (`INT32_LE_BS` / `FLOAT32_LE_BS`).
- The asyn trace on the IP port (`asynSetTraceMask(port, -1, 0x9)`,
  `asynSetTraceIOMask(port, -1, 0x4)`) prints every frame in hex, one
  `write N` header per request; byte 8 is the function code.

### Testing without hardware

geecs-epics `hosts/sim/compose.yaml` (profile `plc`) runs this image
against the Modbus fake on the real `plctest1` config, with the fake's log
as the record of every function code the IOC sent.

## Template Info
This repository was generated by
[ioc-template](https://github.com/epics-containers/ioc-template) 4.8.0
(`copier update -a --trust .` to update). The devcontainer and `./build`
work as in every epics-containers generic IOC; the IOC binary links
`$(SUPPORT)/modbus` through `ibek`.
