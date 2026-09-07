# spat.proto — SPaT (Signal Phase and Timing) for V2X

`spat.proto` carries live signal phase and timing data for V2X connected-vehicle applications. Structured to be directly mappable to SAE J2735 MovementState semantics so that producers can originate, and consumers can relay, compliant SPaT messages without re-translation.

Use case: sub-10ms SPaT broadcast for GLOSA (Green Light Optimal Speed Advisory), signal preemption/priority eligibility checks, and connected-vehicle arrival-timing predictions.

## `SpatUpdate` — live phase state

| Field | # | Type | Notes |
|-------|---|------|-------|
| `intersection_id` | 1 | uint32 | **SAE J2735 DSRC numeric ID** (uint16 range) — NOT the friendly site string in `IntersectionUpdate.intersection_id`. |
| `region_id` | 2 | uint32 | J2735 regional identifier (e.g., 0 = USA). |
| `movements` | 3 | repeated MovementState | One entry per signal group / lane assignment. |
| `sequence_number` | 10 | uint32 | Monotonic; consumers use gaps to detect drops / replays. |
| `update_time` | 11 | google.protobuf.Timestamp | Time this SPaT snapshot was generated. |

## `MovementState` — per-movement state

| Field | # | Type | Notes |
|-------|---|------|-------|
| `signal_group_id` | 1 | uint32 | J2735 signal group (1-255) — cross-ref to MAP message. |
| `movement_name` | 2 | string | Human-readable (e.g., `"NB-Left"`, `"SB-Thru"`). |
| `current_state` | 3 | MovementPhaseState | Current J2735 event + simplified color. |
| `timing` | 4 | TimeChangeDetails | Predicted timing (optional). |
| `nema_phases` | 10 | repeated uint32 | NEMA phase IDs (1-16) this movement maps to — enables SDLC correlation back to ATSPM events. |

## `MovementPhaseState`

| Field | # | Type |
|-------|---|------|
| `event_state` | 1 | MovementEvent |
| `color` | 2 | SignalColor |

## `MovementEvent` enum (J2735)

Vehicular states (1-10) and pedestrian states (11-13):

| Value | Name | Meaning |
|-------|------|---------|
| 0 | `MOVEMENT_EVENT_UNSPECIFIED` | |
| 1 | `UNAVAILABLE` | State unknown |
| 2 | `DARK` | Signal dark (power failure or off) |
| 3 | `STOP_THEN_PROCEED` | Flashing red |
| 4 | `STOP_AND_REMAIN` | Solid red |
| 5 | `PRE_MOVEMENT` | Red clearance (protected turn about to start) |
| 6 | `PERMISSIVE_MOVEMENT_ALLOWED` | Flashing yellow arrow |
| 7 | `PROTECTED_MOVEMENT_ALLOWED` | Green arrow / green ball |
| 8 | `PERMISSIVE_CLEARANCE` | Yellow ball |
| 9 | `PROTECTED_CLEARANCE` | Yellow arrow |
| 10 | `CAUTION_CONFLICTING_TRAFFIC` | Flashing yellow |
| 11 | `PED_WALK` | |
| 12 | `PED_CLEARANCE` | Flashing Don't Walk |
| 13 | `PED_DONT_WALK` | Solid Don't Walk |

## `TimeChangeDetails` — timing predictions

All time fields are milliseconds.

| Field | # | Type | Notes |
|-------|---|------|-------|
| `min_end_time` | 1 | uint32 | Pessimistic — earliest the state can change |
| `likely_end_time` | 2 | uint32 | Best estimate for GLOSA |
| `max_end_time` | 3 | uint32 | Optimistic — latest the state can change |
| `confidence` | 4 | TimingConfidence | Prediction confidence level |
| `next_state` | 5 | MovementEvent | Deterministic next state (optional) |
| `next_duration` | 6 | uint32 | Expected duration of next state (ms) |

`TimingConfidence`: `TIMING_CONFIDENCE_UNSPECIFIED` (producer didn't set it), `TIMING_UNAVAILABLE` (free/adaptive), `TIMING_LOW` (±5s), `TIMING_MEDIUM` (±2s), `TIMING_HIGH` (±500ms, coordinated).

## `SignalColor` — quick-check color

For consumers that don't need full J2735 event semantics. Mirrors the actual signal head output.

Values: `COLOR_DARK`, `COLOR_RED`, `COLOR_YELLOW`, `COLOR_GREEN`, `COLOR_FLASHING_RED`, `COLOR_FLASHING_YELLOW`, `COLOR_FLASHING_GREEN`.

## Wire-format tips

- SPaT is typically broadcast at 10 Hz for V2X. Expect heavy message volume — wrap in `IntersectionUpdateBatch` if pushing through a high-latency transport.
- `nema_phases` is how downstream systems link SPaT (J2735 signal-group-centric) with ATSPM (NEMA-phase-centric) data streams.
- The distinction between `SpatUpdate.intersection_id` (J2735 DSRC uint16) and `IntersectionUpdate.intersection_id` (friendly string) matters: the first is for V2X interoperability, the second is for internal site routing. Producers MUST set both correctly.
