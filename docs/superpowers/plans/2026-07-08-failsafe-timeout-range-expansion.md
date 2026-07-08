# Failsafe Timeout Range Expansion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Widen the declared `max` bounds of `COM_RC_LOSS_T` (manual control loss timeout) and `COM_DL_LOSS_T` (GCS datalink loss timeout) so testers can `param set` values beyond the current firmware-declared limits without a code change per trial.

**Architecture:** Single-file parameter-metadata edit in `src/modules/commander/commander_params.yaml`. No C++ logic changes — the timeout-consuming code (`rcAndDataLinkCheck.cpp`, `Commander.cpp`) reads these params via `.get()` with no additional bound-checking, and PX4 firmware does not enforce `min`/`max` at runtime (`src/lib/parameters/parameters.cpp` has no clamping logic); these bounds are pure metadata read by GCS tooling (e.g. QGroundControl) for UI validation.

**Tech Stack:** PX4 SITL build (`make`/CMake), PX4 parameter-metadata generator, PX4 SITL shell (`pxh>`).

## Global Constraints

- Only `src/modules/commander/commander_params.yaml` may be modified — no other file references these params' bounds (confirmed by source survey in the design spec).
- `default` values stay unchanged: `COM_RC_LOSS_T` default `0.5`, `COM_DL_LOSS_T` default `10`. Do not touch `min`, `decimal`, `increment`, `unit`, or `description` fields.
- `COM_RC_LOSS_T.max`: `35` → `120`.
- `COM_DL_LOSS_T.max`: `300` → `1800`.
- No failsafe *logic* changes (hysteresis, debounce, state machine) — out of scope per the design spec.
- Follow this repo's `CLAUDE.md`: run `make format` on any changed C/C++ before committing (not applicable here — only YAML changes), conventional commit format via the `/commit` skill, no Claude attribution in commits/PRs.

Spec: `docs/superpowers/specs/2026-07-08-failsafe-timeout-range-expansion-design.md`

---

### Task 1: Widen COM_RC_LOSS_T and COM_DL_LOSS_T max bounds

**Files:**
- Modify: `src/modules/commander/commander_params.yaml:14` (`COM_DL_LOSS_T.max`)
- Modify: `src/modules/commander/commander_params.yaml:38` (`COM_RC_LOSS_T.max`)

**Interfaces:**
- Consumes: nothing (only task in this plan).
- Produces: nothing further downstream in this plan. The wider bounds are the deliverable; picking a final `default` value from runtime testing is explicitly out of scope (per spec) and not a follow-up task here.

- [ ] **Step 1: Build the parameter metadata to capture the current (baseline) bounds**

Run:
```bash
make parameters_metadata
```
Expected: build succeeds, producing `build/px4_sitl_default/docs/parameters.md` (or `.xml`) among other outputs, with no errors.

- [ ] **Step 2: Confirm the baseline max values in the generated metadata**

Run:
```bash
grep -A5 '^### COM_DL_LOSS_T' build/px4_sitl_default/docs/parameters.md
grep -A5 '^### COM_RC_LOSS_T' build/px4_sitl_default/docs/parameters.md
```
Expected: output shows `COM_DL_LOSS_T` max `300` and `COM_RC_LOSS_T` max `35` — this documents the pre-change state so Step 5 has something to diff against. (If the generated doc format differs, `grep -n 'COM_DL_LOSS_T\|COM_RC_LOSS_T' -A8 build/px4_sitl_default/docs/parameters.md` is an acceptable fallback — the goal is just confirming 300 and 35 appear before editing.)

- [ ] **Step 3: Edit `COM_DL_LOSS_T.max` in `commander_params.yaml`**

In `src/modules/commander/commander_params.yaml`, change:
```yaml
    COM_DL_LOSS_T:
      description:
        short: GCS connection loss time threshold
        long: After this amount of seconds without datalink, the GCS connection lost
          mode triggers
      type: int32
      default: 10
      unit: s
      min: 5
      max: 300
      decimal: 1
```
to:
```yaml
    COM_DL_LOSS_T:
      description:
        short: GCS connection loss time threshold
        long: After this amount of seconds without datalink, the GCS connection lost
          mode triggers
      type: int32
      default: 10
      unit: s
      min: 5
      max: 1800
      decimal: 1
```
(Only the `max` value changes, from `300` to `1800`.)

- [ ] **Step 4: Edit `COM_RC_LOSS_T.max` in `commander_params.yaml`**

In the same file, change:
```yaml
    COM_RC_LOSS_T:
      description:
        short: Manual control loss timeout
        long: |-
          The time in seconds without a new setpoint from RC or Joystick, after which the connection is considered lost.
          This must be kept short as the vehicle will use the last supplied setpoint until the timeout triggers.
          Ensure the value is not set lower than the update interval of the RC or Joystick.
      type: float
      default: 0.5
      unit: s
      min: 0
      max: 35
      decimal: 1
      increment: 0.1
```
to:
```yaml
    COM_RC_LOSS_T:
      description:
        short: Manual control loss timeout
        long: |-
          The time in seconds without a new setpoint from RC or Joystick, after which the connection is considered lost.
          This must be kept short as the vehicle will use the last supplied setpoint until the timeout triggers.
          Ensure the value is not set lower than the update interval of the RC or Joystick.
      type: float
      default: 0.5
      unit: s
      min: 0
      max: 120
      decimal: 1
      increment: 0.1
```
(Only the `max` value changes, from `35` to `120`.)

- [ ] **Step 5: Rebuild the parameter metadata and confirm the new bounds are picked up**

Run:
```bash
make parameters_metadata
grep -A5 '^### COM_DL_LOSS_T' build/px4_sitl_default/docs/parameters.md
grep -A5 '^### COM_RC_LOSS_T' build/px4_sitl_default/docs/parameters.md
```
Expected: build succeeds (a schema error in the YAML edit would fail this step), and the output now shows `COM_DL_LOSS_T` max `1800` and `COM_RC_LOSS_T` max `120`.

- [ ] **Step 6: Boot headless SITL and verify the new bounds are usable at runtime**

Run:
```bash
make px4_sitl none_iris
```
Expected: builds and drops into the `pxh>` shell (no simulator client needed for `none_iris`).

In the `pxh>` shell:
```bash
param set COM_DL_LOSS_T 900
param show COM_DL_LOSS_T
param set COM_RC_LOSS_T 100
param show COM_RC_LOSS_T
```
Expected: both `param set` calls succeed (previously `900` and `100` were outside the declared `300`/`35` max), and `param show` reports the new values `900` and `100` respectively. Exit the shell with `shutdown`.

- [ ] **Step 7: Commit**

```bash
git add src/modules/commander/commander_params.yaml
git commit -s -m "feat(commander): widen COM_RC_LOSS_T and COM_DL_LOSS_T max bounds" -m "Testing failsafe activation timing needs values beyond the current
35s/300s ceilings. Firmware does not enforce min/max at runtime (only
GCS tooling does via this metadata), so raising the declared max lets
testers iterate on timeout values with plain param set — no rebuild
per trial. Defaults are unchanged; picking a final value is a
follow-up once testing is done."
```
Expected: commit succeeds with only `commander_params.yaml` staged. (If still on `main`, first run `git checkout -b <username>/failsafe-timeout-range` per the `/commit` skill — this work was already branched as `ThangDuc3101/failsafe-timeout-range-spec` when the design spec was committed; continue on that branch if it's still checked out.)

- [ ] **Step 8: Open the PR**

Invoke the repo's `/pr` skill to open a pull request from the current branch, using the design spec (`docs/superpowers/specs/2026-07-08-failsafe-timeout-range-expansion-design.md`) for the PR description's context. Do not add Claude attribution (per `CLAUDE.md`).
