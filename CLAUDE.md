# ioc-plc

The PLC and digital-I/O image of GEECS-BELLA's native EPICS IOCs:
epics-containers' `epics-base` developer image plus asyn R4-46 and
`epics-modules/modbus` R3-4, both from upstream ibek-support recipes,
unchanged. What the driver sends at start, and why a start is read-only:
`README.md`.

- **Instances** live in GEECS-BELLA/geecs-epics, not here: `plctest1`
  (Productivity1000 `U_PLC`, `hosts/htu-linux-camserv2`, profile
  `bench-plc`) and its fake twin `plctest1-sim` (`hosts/sim`, profile
  `plc`). Records come from the instance's own templates and substitutions,
  not from this image.
- **Why ours.** No epics-containers runtime image carries modbus (checked
  2026-10-24). No patch: a write port with a non-zero poll delay reads its
  block once at creation and the output records start from that read, so a
  start sends reads only (proven on the fake and twice on the PLC).
- **Build.** On the IOC host: `docker build --target runtime -t ioc-plc:local ~/ioc-plc`.
  On a workstation: `DOCKER_DEFAULT_PLATFORM=linux/amd64 TARGET=runtime
  TAG=ioc-plc:local ./build` (the variable is required on Apple silicon; the
  image is amd64 and runs emulated).
- **Pins.** geecs-epics' compose file notes the commit the host image was
  built from. A bump: rebuild on the host (never during a run), update that
  note in the same change, recreate the container (`docker restart` keeps
  the old image), one IOC on the bench first.
- **Merging**: after a fresh-context review dispositioned on the PR, and the
  bench proof for anything that changes what reaches a device. Rules and
  program state: geecs-epics `CLAUDE.md`.
