# OpenPhase v1 — Protocol Overview

OpenPhase v1 is a Protobuf-based open protocol for traffic-signal telemetry, control, and analytics. It carries everything a modern signal infrastructure needs to publish or accept over a NATS JetStream / MQTT / gRPC transport: live phase state, high-resolution event logs, NTCIP MIB objects, device health, security alerts, and fault snapshots.

All messages live in the `openphase.v1` package. Every `.proto` file in `openphase/v1/` is a self-contained schema domain.

## Schemas at a glance

| File | Domain | Wraps | Purpose |
|------|--------|-------|---------|
| [common.proto](common.md) | Transport wrapper | All payloads | `IntersectionUpdate` envelope (oneof discriminator), batch wrappers, `ClockSource` |
| [ihr_events.proto](ihr_events.md) | ATSPM events | Streamed via wrapper | Indiana Hi-Resolution event log (`AtspmEvent`, `EventType` enum, `CompactEvent` for batches) |
| [spat.proto](spat.md) | V2X / GLOSA | Streamed via wrapper | Live phase state per SAE J2735 (`SpatUpdate`, `MovementState`) |
| [ntcip.proto](ntcip.md) | NTCIP 1202 v03A | Bulk via wrapper | Full MIB-equivalent OID-addressed messages for ASC config & status |
| [health.proto](health.md) | Telemetry | 60s heartbeat | `DeviceHealth` (resources, comm stats, environmental) |
| [security.proto](security.md) | Alerts | On-event | `SecurityAlert` (spoof detection, MMU conflict, intrusion) |
| [discovery.proto](discovery.md) | Self-learning | On-event | `DiscoveryPacket` for unmapped SDLC bit transitions |
| [faults.proto](faults.md) | Snapshots | On-fault | `FaultSnapshot` capturing event history + state at fault time |

## Carrier model

Every per-event payload travels inside `IntersectionUpdate` (in `common.proto`):

```proto
message IntersectionUpdate {
  string intersection_id = 1;
  google.protobuf.Timestamp ts = 2;
  oneof payload {
    SpatUpdate spat = 3;
    AtspmEvent event = 4;
    SecurityAlert security = 5;
    DiscoveryPacket discovery = 6;
    DeviceHealth health = 7;
    FaultSnapshot fault = 8;
    ControllerConfig config = 9;
    ControllerStatus status = 10;
  }
  ClockSource clock_source = 15;
}
```

Two batch wrappers exist for transport efficiency:
- **`IntersectionUpdateBatch`** — mixed-payload batches (each row is a full `IntersectionUpdate`).
- **`CompactEventBatch`** — homogeneous IHR-event batches with a shared base timestamp and per-event `offset_ms` deltas. Use for high-volume ATSPM streaming over bandwidth-constrained backhaul.

## Standards and references

- **Indiana Hi-Resolution Data Logger Enumerations** — Sturdevant et al., INDOT/Purdue, November 2012, [doi:10.4231/K4RN35SH](https://doi.org/10.4231/K4RN35SH). Drives `EventType` in `ihr_events.proto`.
- **NTCIP 1202 v03A** — Object Definitions for Actuated Signal Controllers, May 2019. Drives the OID-addressed messages in `ntcip.proto`.
- **SAE J2735** — Movement event states for V2X. Drives `MovementEvent` in `spat.proto`.
- **NEMA TS 2** — Referenced throughout NTCIP 1202 for parameter semantics.

## Versioning

`openphase.v1` is the current stable version. Per the breaking-change policy in the [README](../README.md), additions are non-breaking; field renumbering or semantic changes require a new version folder (`openphase/v2/`). Modified protocol files must be made available under MPL 2.0.
