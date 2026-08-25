# Mission Manager engineering TODO

## P0 — integration blockers

- [ ] Add a joint test with `omni_robot_bridge` proving acquire, renew,
  preemption and release through typed `ControlAuthority`.
- [ ] Gate mission dispatch on `/omni/tf_manager/ready` and validate the
  `omni_map -> omni_base_link` transform at the goal timestamp.
- [ ] Replace the legacy pose/route frame defaults only after a compatibility
  migration test covers existing `lio_map` route files.
- [ ] Add launch/integration tests with SCAN-Planner for accept, feedback,
  cancel, planner death and stale feedback.
- [ ] Resolve the extracted-source license conflict and add the authoritative
  root `LICENSE`.

## P1 — production hardening

- [ ] Add lifecycle/readiness reporting and systemd watchdog integration.
- [ ] Define SQLite schema migrations, backup/restore and disk-full behavior.
- [ ] Add bounded event/history retention and evidence storage quotas.
- [ ] Add DDS security/QoS soak tests for restart, packet loss and delayed
  transient-local samples.
- [ ] Add metrics for dispatch rejection, lease loss, planner latency,
  checkpoint duration and recovery outcome.
- [ ] Add fault-injection tests for process kill at every durable state
  transition.

## P2 — operations

- [ ] Publish signed Debian/OTA artifacts for x86_64, Orin and S100.
- [ ] Version the route/checkpoint schemas and provide an offline validator.
- [ ] Provide operator runbooks for interrupted missions, database repair and
  manual return-to-dock recovery.
