# ihr_events.proto — Indiana Hi-Resolution Event Logging

`ihr_events.proto` carries the ATSPM "nervous system" — high-resolution event log entries from a traffic-signal controller. Volume is roughly 2 million events/day per intersection. The schema is the canonical Indiana Hi-Resolution Data Logger Enumerations (Sturdevant et al., INDOT/Purdue, November 2012, [doi:10.4231/K4RN35SH](https://doi.org/10.4231/K4RN35SH)) — the same enumeration used by every ATSPM-compatible controller.

## `AtspmEvent` — single event

Per the IHR spec, an event-log entry is exactly two 8-bit fields: Event Code and Parameter. The intersection identifier and timestamp are carried by the outer envelope (`IntersectionUpdate.intersection_id` and `.ts`), not in `AtspmEvent` itself.

| Field | # | Type | Notes |
|-------|---|------|-------|
| `code` | 1 | EventType | IHR Event Code (0-255). Enum integer == IHR code. |
| `param` | 2 | uint32 | IHR Parameter (0-255). Semantics depend on `code` — phase #, detector #, preempt #, overlap #, etc. |

Producers MUST constrain `param` to 0-255 (protobuf has no uint8; varint-encoded 0-127 = 1 byte, 128-255 = 2 bytes).

For Event Codes outside the standard `EventType` enumeration (vendor extensions in 200-255, or future spec additions), set `code = EVENT_TYPE_UNSPECIFIED` and put the raw integer in a wrapping `CompactEvent` or use the integer field number directly via `extended_event_code` if extending; this proto currently does not include an extended-code escape hatch on `AtspmEvent` itself — use the IHR enum.

## `CompactEvent` — batched event with relative timing

Used inside `CompactEventBatch` (in `common.proto`) and `FaultSnapshot.recent_events` (in `faults.proto`). Adds a millisecond offset from a shared base timestamp.

| Field | # | Type | Notes |
|-------|---|------|-------|
| `offset_ms` | 1 | uint32 | Milliseconds from the batch's `base_timestamp_ns` (varint, typically 1-2 bytes) |
| `code` | 2 | EventType | Same as `AtspmEvent.code` |
| `param` | 3 | uint32 | Same as `AtspmEvent.param` |

Absolute time = `base_timestamp_ns + (offset_ms * 1_000_000)`.

## `EventType` enum

The integer value of every `EventType` constant equals its Indiana Hi-Res event code. No translation table is needed at decode time. Code 0 ("Phase On NEMA") is intentionally omitted to preserve the proto3 zero-value as `EVENT_TYPE_UNSPECIFIED`; producers needing Phase On NEMA semantics should emit Phase Begin Green (1) instead.

### Active Phase Events (1-12)
`param` = phase number (1-16).

| Code | Name | Meaning |
|------|------|---------|
| 1 | `EVENT_PHASE_BEGIN_GREEN` | Solid or flashing green begins |
| 2 | `EVENT_PHASE_CHECK` | Conflicting call registered (begins MAX timing) |
| 3 | `EVENT_PHASE_MIN_COMPLETE` | Minimum-green timer expired |
| 4 | `EVENT_PHASE_GAP_OUT` | Phase gapped out |
| 5 | `EVENT_PHASE_MAX_OUT` | Phase MAX timer expired |
| 6 | `EVENT_PHASE_FORCE_OFF` | Force-off applied to active green phase |
| 7 | `EVENT_PHASE_GREEN_TERMINATION` | Green terminated into yellow or permissive (FYA) |
| 8 | `EVENT_PHASE_BEGIN_YELLOW_CLEARANCE` | Yellow clearance begins |
| 9 | `EVENT_PHASE_END_YELLOW_CLEARANCE` | Yellow clearance ends |
| 10 | `EVENT_PHASE_BEGIN_RED_CLEARANCE` | Red clearance begins (only set if served) |
| 11 | `EVENT_PHASE_END_RED_CLEARANCE` | Red clearance ends |
| 12 | `EVENT_PHASE_INACTIVE` | Phase no longer active in ring |

### Pedestrian Events (21-24)
`param` = phase number.

| Code | Name |
|------|------|
| 21 | `EVENT_PED_BEGIN_WALK` |
| 22 | `EVENT_PED_BEGIN_CLEARANCE` (FDW) |
| 23 | `EVENT_PED_BEGIN_SOLID_DONT_WALK` |
| 24 | `EVENT_PED_DARK` |

### Barrier / Ring Events (31-33)
| Code | Name | param |
|------|------|-------|
| 31 | `EVENT_BARRIER_TERMINATION` | barrier # (1-8) |
| 32 | `EVENT_FYA_BEGIN_PERMISSIVE` | FYA # (1-4) |
| 33 | `EVENT_FYA_END_PERMISSIVE` | FYA # (1-4) |

### Phase Control Events (41-49)
`param` = phase number.

`EVENT_PHASE_HOLD_ACTIVE`/`_RELEASED`, `EVENT_PHASE_CALL_REGISTERED`/`_DROPPED`, `EVENT_PED_CALL_REGISTERED`, `EVENT_PHASE_OMIT_ON`/`_OFF`, `EVENT_PED_OMIT_ON`/`_OFF`.

### Overlap Events (61-70)
`param` = overlap number (A=1, B=2, …).

`EVENT_OVERLAP_BEGIN_GREEN`, `_BEGIN_TRAILING_GREEN`, `_BEGIN_YELLOW`, `_BEGIN_RED_CLEARANCE`, `_OFF`, `_DARK`, plus `EVENT_PED_OVERLAP_*` (67-70).

### Detector Events (81-92)
`param` = detector channel (1-64 vehicle, 1-16 pedestrian). Emitted POST detector delay/extension processing.

| Code | Name |
|------|------|
| 81 | `EVENT_DETECTOR_OFF` |
| 82 | `EVENT_DETECTOR_ON` |
| 83 | `EVENT_DETECTOR_RESTORED` |
| 84 | `EVENT_DETECTOR_FAULT_OTHER` |
| 85 | `EVENT_DETECTOR_FAULT_WATCHDOG` |
| 86 | `EVENT_DETECTOR_FAULT_OPEN_LOOP` |
| 87 | `EVENT_DETECTOR_FAULT_SHORTED_LOOP` |
| 88 | `EVENT_DETECTOR_FAULT_EXCESSIVE_CHANGE` |
| 89 | `EVENT_PED_DETECTOR_OFF` |
| 90 | `EVENT_PED_DETECTOR_ON` |
| 91 | `EVENT_PED_DETECTOR_FAILED` |
| 92 | `EVENT_PED_DETECTOR_RESTORED` |

### Preemption + TSP Events (101-115)
`param` = preempt # (1-10) for 101-111, TSP # (1-10) for 112-115.

| Code | Name |
|------|------|
| 101 | `EVENT_PREEMPT_ADVANCE_WARNING` |
| 102 | `EVENT_PREEMPT_CALL_INPUT_ON` (request begin) |
| 103 | `EVENT_PREEMPT_GATE_DOWN_INPUT` |
| 104 | `EVENT_PREEMPT_CALL_INPUT_OFF` (request end) |
| 105 | `EVENT_PREEMPT_ENTRY_STARTED` (service begin) |
| 106 | `EVENT_PREEMPT_BEGIN_TRACK_CLEARANCE` |
| 107 | `EVENT_PREEMPT_BEGIN_DWELL_SERVICE` |
| 108 | `EVENT_PREEMPT_LINK_ACTIVE_ON` |
| 109 | `EVENT_PREEMPT_LINK_ACTIVE_OFF` |
| 110 | `EVENT_PREEMPT_MAX_PRESENCE_EXCEEDED` |
| 111 | `EVENT_PREEMPT_BEGIN_EXIT_INTERVAL` (service end) |
| 112 | `EVENT_TSP_CHECK_IN` |
| 113 | `EVENT_TSP_ADJUSTMENT_TO_EARLY_GREEN` |
| 114 | `EVENT_TSP_ADJUSTMENT_TO_EXTEND_GREEN` |
| 115 | `EVENT_TSP_CHECK_OUT` |

**Common pairings:**
- Preempt request duration: 102 → 104
- Preempt service duration: 105 → 111
- TSP request lifetime: 112 → 115

### Coordination Events (131-151)

| Code | Name | param semantics |
|------|------|-----------------|
| 131 | `EVENT_COORD_PATTERN_CHANGE` | pattern # (0-255) |
| 132 | `EVENT_COORD_CYCLE_LENGTH_CHANGE` | seconds |
| 133 | `EVENT_COORD_OFFSET_LENGTH_CHANGE` | seconds |
| 134-149 | `EVENT_COORD_SPLIT_N_CHANGE` (N=1..16) | new split seconds |
| 150 | `EVENT_COORD_CYCLE_STATE_CHANGE` | `CoordCycleState` enum value |
| 151 | `EVENT_COORD_PHASE_YIELD_POINT` | phase # |

`CoordCycleState` enum (param values for event 150): `COORD_CYCLE_STATE_FREE` (0), `COORD_CYCLE_STATE_IN_STEP` (1), `COORD_CYCLE_STATE_TRANSITION_ADD` (2), `COORD_CYCLE_STATE_TRANSITION_SUBTRACT` (3), `COORD_CYCLE_STATE_TRANSITION_DWELL` (4), `COORD_CYCLE_STATE_LOCAL_ZERO` (5), `COORD_CYCLE_STATE_BEGIN_PICKUP` (6).

### Cabinet / System Events (171-185)

| Code | Name | param |
|------|------|-------|
| 171 | `EVENT_TEST_INPUT_ON` | test input # |
| 172 | `EVENT_TEST_INPUT_OFF` | test input # |
| 173 | `EVENT_UNIT_FLASH_STATUS_CHANGE` | NTCIP flash state |
| 174 | `EVENT_UNIT_ALARM_STATUS_1_CHANGE` | NTCIP alarm state |
| 175 | `EVENT_ALARM_GROUP_STATE_CHANGE` | NTCIP group state |
| 176 | `EVENT_SPECIAL_FUNCTION_OUTPUT_ON` | function # |
| 177 | `EVENT_SPECIAL_FUNCTION_OUTPUT_OFF` | function # |
| 178 | `EVENT_MANUAL_CONTROL_TOGGLE` | 0/1 |
| 179 | `EVENT_INTERVAL_ADVANCE_TOGGLE` | 0/1 |
| 180 | `EVENT_STOP_TIME_INPUT_TOGGLE` | 0/1 |
| 181 | `EVENT_CONTROLLER_CLOCK_UPDATED` | correction in seconds (optional) |
| 182 | `EVENT_POWER_FAILURE_DETECTED` | — |
| 184 | `EVENT_POWER_RESTORED` | — |
| 185 | `EVENT_VENDOR_SPECIFIC_ALARM` | vendor-defined |

(183 intentionally absent in the spec.)

## NTCIP relation

NTCIP 1202 is OID-addressed via SNMP and does NOT define an event-code enumeration. The Indiana Hi-Res enumeration above is the de-facto event-log standard used by ATSPM. NTCIP MIB objects (phase config, status groups, etc.) are modeled separately in [ntcip.proto](ntcip.md).
