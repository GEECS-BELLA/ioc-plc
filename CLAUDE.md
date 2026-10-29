# ioc-plc

The PLC and digital-I/O image of GEECS-BELLA's native EPICS IOCs:
epics-containers' `epics-base` developer image plus asyn R4-46 and
`epics-modules/modbus` R3-4, both from upstream ibek-support recipes,
unchanged. What the driver sends at start, and why a start is read-only:
`README.md`.

- **Instances** live in GEECS-BELLA/geecs-epics, not here: `plctest1`
  (Productivity1000 `U_PLC`, profile `bench-plc`) and its fake twin
  `plctest1-sim` (`hosts/sim`, profile `plc`). Records come from the
  instance's own templates and substitutions, not from this image.
- **Why ours.** No epics-containers runtime image carries modbus (checked
  2026-10-24). No patch: a write port with a non-zero poll delay reads its
  block once at creation and the output records start from that read, so an
  instance whose output records have no `PINI` and no autosave starts with
  reads only (README; proven on the fake and twice on the PLC, geecs-epics
  #16's phase-A slice record).
- **Images.** CI publishes on a git tag, CalVer `YYYY.M.N` (`2026.10.1`;
  never re-push a tag): `ghcr.io/geecs-bella/ioc-plc-runtime:<tag>` (what hosts
  pull, no credentials) and `-developer` (the release job runs in it; a build
  base); both packages made public once, after the first tag creates them. Hosts never build. Bench work from a branch: a
  pre-release tag (`2026.10.2b1`). A local build for development:
  `DOCKER_DEFAULT_PLATFORM=linux/amd64 TARGET=runtime TAG=ioc-plc:local ./build`
  (the variable is required on Apple silicon; the image is amd64, emulated).
- **Pins.** geecs-epics' compose files pin the tag. A bump: tag here, change
  the tag in compose in one geecs-epics PR, `docker compose pull <service>`
  and recreate on the host (`docker restart` keeps the old image), one IOC
  on the bench first. Rollback: revert that compose change and recreate.
- **Merging**: after a fresh-context review dispositioned on the PR, and the
  bench proof for anything that changes what reaches a device. Rules and
  program state: geecs-epics `CLAUDE.md`.
