# Mission Manager interface contract

## Responsibility boundary

`omni_mission_manager` owns mission state, idempotency, route/checkpoint
execution, event persistence and the return-to-dock orchestration chain. It
does not calculate localization, generate trajectories, arbitrate velocity,
talk to a vendor SDK or implement final docking control.

## Provided interfaces

| Kind | Default name | Type | Semantics |
|---|---|---|---|
| Action | `/omni/mission/execute` | `ExecuteInspection` | Long-running inspection mission |
| Action | `/omni/mission/return_to_dock` | `ReturnToDock` | Global approach followed by Dock action handoff |
| Service | `/omni/mission/dispatch` | `DispatchMission` | Service-compatible dispatch entry with the same gates as the action |
| Service | `/omni/mission/control` | `MissionControl` | Pause, resume and cancel |
| Service | `/omni/routes/list` | `ListRoutes` | Enumerate validated route files |
| Service | `/omni/mission/results` | `GetCheckpointResults` | Durable checkpoint evidence query |
| Topic | `/omni/mission/status` | `MissionStatus` | Reliable/transient-local latest state |
| Topic | `/omni/mission/events` | `MissionEvent` | Reliable append-only event stream |
| Topic | `/omni/mission/checkpoint_results` | `CheckpointResult` | Reliable/transient-local evidence result |

## Consumed interfaces

| Kind | Default name | Type | Required behavior |
|---|---|---|---|
| Action | `/omni/navigation/follow_route` | `FollowRoute` | Planner accepts unique leg IDs and supports cancel |
| Action | `/omni/docking/dock` | `Dock` | Docking owns the DOCKING lease during final approach |
| Service | `/omni/docking/config` | `GetDockConfig` | Resolves a map/version to a dock pose |
| Service | `/omni/control/authority` | `ControlAuthority` | Acquire/renew/release MISSION authority |
| Services | `/omni/capture/photo`, `/omni/capture/record`, `/omni/recognize` | `CapturePhoto`, `StartRecord`, `Recognize` | Bounded checkpoint operations |
| Topic | `/omni/robot_state` | `RobotState` | Reliable/transient-local aggregate health and localization gate |
| Topic | `/state_estimation_global` | `nav_msgs/Odometry` | Current map-frame pose for return-to-dock approach path |

All custom types come from `omni_robot_interfaces`. `RobotState` is assembled
by the robot bridge from SLAM, battery, adapter, E-stop and mission inputs.

## Cross-repository sequence

```text
App/gateway -> mission manager: DispatchMission / ExecuteInspection
mission manager -> robot bridge: acquire MISSION authority
mission manager -> SCAN-Planner: FollowRoute
SCAN-Planner -> robot bridge: /scan_planner/cmd_vel
mission manager -> omni_docking: Dock
omni_docking -> robot bridge: acquire DOCKING authority + docking cmd_vel
robot bridge -> vendor SDK: one safety-clamped final velocity stream
```

The authority handoff must be fail-closed: MISSION is released before the
Dock goal is sent, and Docking must stop publishing a non-zero command before
releasing DOCKING.

## Compatibility issues to close

- `ControlAuthority` is consumed here but the extracted bridge currently only
  implements the legacy `/rosdeck/control_command` string protocol. This is a
  P0 integration gap, not an optional enhancement.
- Pose defaults and legacy route headers still use
  `/state_estimation_global`/`lio_map`; the target navigation contract uses
  `omni_map -> omni_odom -> omni_base_link`. Migration needs aliases plus
  cross-repository tests before defaults change.
- The App-facing gateway is a client of these typed interfaces. It is not the
  authority owner and must not publish directly to the SDK-facing velocity
  topic.
