# omni_mission_manager

Robot-side mission orchestrator for the omni inspection robot. It is the server of the
`ExecuteInspection` action and the provider of `MissionControl` /
`ListRoutes`; it drives the SCAN planner through the `FollowRoute` action
and holds the **MISSION** control lease against the unified robot bridge.

The V1 contract lives in [`YanYaoyuan/omni_robot_interfaces`](https://github.com/YanYaoyuan/omni_robot_interfaces);
constant values below are mirrored locally (see `constants.py`) and pinned
by the interfaces repo CI.

This repository was history-preservingly extracted from `rosdeck/robot`.
It is a robot runtime component and has no dependency on the Rosdeck App.
See [the complete interface contract](docs/INTERFACES.md) and
[the engineering backlog](docs/TODO.md).

## ROS surface

| Kind | Name | Counterparty |
|---|---|---|
| Action (server) | `/omni/mission/execute` (`ExecuteInspection`) | App (rosdeck) |
| Action (client) | `/omni/navigation/follow_route` (`FollowRoute`) | scan planner |
| Service | `/omni/mission/control` (`MissionControl`) | App |
| Service | `/omni/routes/list` (`ListRoutes`) | App |
| Service | `/omni/mission/dispatch` (`DispatchMission`) | App/gateway compatibility entry |
| Service | `/omni/mission/results` (`GetCheckpointResults`) | App |
| Action (server) | `/omni/mission/return_to_dock` (`ReturnToDock`) | App, low-battery watchdog |
| Action (client) | `/omni/docking/dock` (`Dock`) | omni_docking |
| Service (client) | `/omni/docking/config` (`GetDockConfig`) | omni_docking |
| Service (client) | `/omni/control/authority` (`ControlAuthority`) | omni_robot_bridge (target contract) |
| Topic (sub) | `/omni/robot_state` (`RobotState`, transient_local) | omni_robot_bridge |
| Topic (sub) | `/state_estimation_global` (`Odometry`) | omni_slam compatibility output |
| Topic (pub) | `/omni/mission/status` (`MissionStatus`, transient_local) | App, omni_robot_bridge |
| Topic (pub) | `/omni/mission/events` (`MissionEvent`, reliable) | App, diagnostics |
| Topic (pub) | `/omni/mission/checkpoint_results` (`CheckpointResult`) | App, evidence pipeline |

## Lifecycle

```
PENDING --planner EXECUTING feedback--> EXECUTING
PENDING/EXECUTING --pause--> PAUSED --resume--> EXECUTING
* --cancel--> CANCELED
* --planner success--> SUCCEEDED
* --planner failure/lost--> FAILED
active --restart / stop / lease lost--> INTERRUPTED   (never auto-resumed)
```

- **Pause** releases the mission lease; the gateway arbiter then outputs
  zero velocity and the robot stops cleanly. **Resume** re-acquires the
  lease. There is no action-protocol pause primitive.
- **Cancel** is always accepted for the active mission; the planner goal
  is canceled (controlled stop, not estop) and the lease released.
- **Precondition gates** on dispatch (fail fast, reason_code set):
  checkpoint ids must be empty in V1; route exists and is readable;
  map/version consistency (goal vs route binding vs the robot's
  localization); robot state fresh; localization `LOCALIZED`;
  then idempotency / single-active-mission checks.
- **Idempotency**: `(request_id, sequence)` is the key. Same key →
  duplicate, the original result is returned without re-dispatching.
  Higher sequence with the same `request_id` → the old attempt is
  superseded (CANCELED, `superseded by sequence N`) and the new one
  proceeds. Lower sequence → REJECTED as stale.
- A dispatch aborted *before* the `DISPATCHED` event (planner
  unavailable / authority denied) is rolled back and its
  `(request_id, sequence)` key is freed for retry.
- **Lease**: 5 s, renewed every 1 s. Preemption (App takeover) or 3
  consecutive failed renewals interrupt the mission (INTERRUPTED).
- **Planner liveness**: a FollowRoute goal that goes terminal without a
  usable result, or an EXECUTING mission with no planner feedback for 30
  s, fails the mission.
- **Restart recovery**: rows still active in SQLite at startup become
  INTERRUPTED (event recorded). Missions are **never** auto-resumed;
  a clean SIGTERM does the same interruption and releases leases.

## Persistence

SQLite at `database` (default `/var/lib/omni/mission_manager/missions.db`):

- `missions` — one row per dispatched mission (terminal rows kept)
- `mission_events` — append-only event stream, PK `(mission_id, sequence)`,
  sequence starts at 1; the same events are published on
  `/omni/mission/events`
- `idempotency` — `(request_id, sequence) → mission_id`

Routes are read from `routes_dir` (default `/var/lib/omni/routes`) in the
record_path.py format (`# omni_slam global body path v1` header + `x y z`
rows, `lio_map` frame). V1 route files are **unbound** (`map_id=""`);
goal `map_id`/`map_version` empty means "current".

## Build & test

```bash
# core unit tests (pure Python, no ROS needed):
python3 -m unittest discover -s test -v

# ROS workspace build (omni_robot_interfaces must be in the same workspace):
colcon build --packages-up-to omni_mission_manager
```

Deployment ownership is intentionally outside this repository. The product
composition layer should install and supervise this package independently
from the bridge so either component can be rolled back without replacing the
other.

## Parameters

| Parameter | Default |
|---|---|
| `routes_dir` | `/var/lib/omni/routes` |
| `database` | `/var/lib/omni/mission_manager/missions.db` |
| `robot_state_topic` | `/omni/robot_state` |
| `status_topic` | `/omni/mission/status` |
| `events_topic` | `/omni/mission/events` |
| `execute_action` | `/omni/mission/execute` |
| `follow_route_action` | `/omni/navigation/follow_route` |
| `control_service` | `/omni/mission/control` |
| `routes_service` | `/omni/routes/list` |
| `authority_service` | `/omni/control/authority` |
| `lease_sec` | `5.0` |
| `lease_renew_period_sec` | `1.0` |
| `robot_state_stale_ms` | `2000.0` |
| `planner_stale_sec` | `30.0` |

## Repository policy

- ROS interface definitions are consumed from `omni_robot_interfaces`; this
  repository does not duplicate `.msg`, `.srv` or `.action` files.
- Topic/service/action names are parameters, while the defaults above are the
  compatibility contract covered by integration tests.
- The extracted package metadata says `Apache-2.0`, while the former monorepo
  root carried `GPL-3.0`. The owner must resolve that provenance and add an
  authoritative root `LICENSE` before any external binary/source release.
