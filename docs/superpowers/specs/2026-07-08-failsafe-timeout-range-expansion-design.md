# Failsafe Timeout Range Expansion — Design

## Problem

The user needs to experiment with RC-loss and GCS-datalink-loss failsafe
activation timing during testing, trying several timeout values before
settling on a final number. The parameters that control this
(`COM_RC_LOSS_T`, `COM_DL_LOSS_T`) already exist and are read at runtime —
no timing *logic* needs to change. The blocker is that their declared
`max` bounds are too narrow, so QGroundControl (and any other GCS/tooling
that reads PX4 parameter metadata) refuses to accept values the user wants
to try during testing.

Firmware itself does not enforce `min`/`max` when a parameter is set
(confirmed: `src/lib/parameters/parameters.cpp` contains no min/max
clamping logic) — these bounds are pure metadata consumed by GCS software
for UI validation. Widening them in the parameter definition is therefore
sufficient to unblock testing; the user can then iterate on the actual
timeout value at runtime via `param set`, without further code changes or
rebuilds per trial.

## Current state (verified in source)

File: `src/modules/commander/commander_params.yaml`

| Param | Description | default | unit | min | max |
|---|---|---|---|---|---|
| `COM_RC_LOSS_T` | Manual control (RC/Joystick) loss timeout | 0.5 | s | 0 | 35 |
| `COM_DL_LOSS_T` | GCS connection (datalink) loss timeout | 10 | s | 5 | 300 |

Consumers (unchanged by this work):
- `COM_RC_LOSS_T` → `src/modules/commander/HealthAndArmingChecks/checks/rcAndDataLinkCheck.cpp:49`
- `COM_DL_LOSS_T` → `src/modules/commander/Commander.cpp:3028-3032`

Both are read via `hrt_elapsed_time(...) > _param_*.get() * 1_s` with no
additional bound-checking in the consuming code.

## Design

Scope: a single-file metadata change, no C++ logic touched.

- `src/modules/commander/commander_params.yaml`
  - `COM_RC_LOSS_T.max`: `35` → `120` (keep `min: 0`, `default: 0.5`,
    `decimal: 1`, `increment: 0.1`)
  - `COM_DL_LOSS_T.max`: `300` → `1800` (keep `min: 5`, `default: 10`)

`default` values are intentionally left unchanged. The user will tune the
actual operating value at runtime (`param set`) within the new, wider
range; only once a final number is chosen would a follow-up change update
`default` — that is explicitly out of scope here.

No other files reference these params' min/max (confirmed via source
survey), so no other edits are required.

## Testing

1. Build SITL (`make px4_sitl`) — the PX4 param-metadata generator
   validates the yaml schema at build time; a malformed edit fails the
   build.
2. Boot SITL, run `param show COM_RC_LOSS_T` and `param show COM_DL_LOSS_T`
   to confirm the new `max` values are reflected in the compiled metadata.
3. `param set COM_RC_LOSS_T 100` and `param set COM_DL_LOSS_T 900` to
   confirm values previously outside the old bounds are now accepted.
4. `make check_format` / `make format` per project convention before
   commit (CLAUDE.md).

## Out of scope

- Changing `default` values (deferred until the user finishes testing and
  picks a final number).
- Any change to failsafe *logic* (hysteresis, debounce, state machine).
- Other failsafe timing params surfaced during the source survey
  (`COM_HLDL_LOSS_T`, `COM_OF_LOSS_T`, `COM_FAIL_ACT_T`,
  `FD_FAIL_R_TTRI`/`FD_FAIL_P_TTRI`, `FD_ALT_LOSS_T`, EKF/geofence
  timeouts) — not requested by the user.
- GCS-side (QGroundControl) verification — out of reach in this
  environment; SITL `param show`/`param set` is the verification
  substitute.

## Delivery process

Commit and PR via this repo's `/commit` and `/pr` skills (conventional
commit format, topic-based scope, no Claude attribution per CLAUDE.md).
