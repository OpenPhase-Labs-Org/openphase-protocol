# ingestion.proto — gRPC Ingestion Service

`ingestion.proto` defines the server-side gRPC contract for accepting OpenPhase telemetry pushed by field devices. It is the one schema domain in `openphase/v1/` that defines **RPCs rather than payloads**: every other file describes a message that travels inside `IntersectionUpdate`, while this one describes how a device hands those messages to a server that is listening for them.

It is transport-agnostic in the sense that matters — the `IntersectionUpdate` and `CompactEventBatch` messages defined in [common.proto](common.md) travel here **unchanged**. A device that already publishes to NATS JetStream or MQTT sends the identical bytes over gRPC; only the carrier differs. Nothing in this file redefines a payload.

## `IngestionService`

| RPC | Request | Response | Use when |
|-----|---------|----------|----------|
| `PublishUpdate` | `IntersectionUpdate` | `PublishAck` | Low-volume or mixed-payload publishing where each message stands alone — a health heartbeat, a security alert, a fault snapshot |
| `PublishBatch` | `CompactEventBatch` | `PublishAck` | Medium-volume periodic pushes of IHR events, on the order of every few seconds |
| `StreamBatches` | `stream CompactEventBatch` | `PublishAck` | High-volume continuous telemetry — the common ATSPM streaming case. The device holds the stream open and pushes batches as they are produced |

`PublishUpdate` accepts any payload the `IntersectionUpdate` oneof carries: SPaT, IHR event, health, security, discovery, fault, controller config and controller status. `PublishBatch` and `StreamBatches` are IHR-event paths only, since `CompactEventBatch` is homogeneous by construction.

## `PublishAck` — acknowledgement

| Field | # | Type | Notes |
|-------|---|------|-------|
| `events_accepted` | 1 | uint32 | Number of events successfully decoded and persisted |
| `error` | 2 | string | Empty on success; human-readable cause otherwise |

## Acknowledgement semantics

`PublishUpdate` and `PublishBatch` return one `PublishAck` per call. `StreamBatches` returns a **single** `PublishAck` when the stream closes, and its `events_accepted` is the **cumulative** count across every batch sent on that stream — not a per-batch acknowledgement.

That difference matters for a producer's retry design. On the unary RPCs a device learns the fate of each push immediately. On `StreamBatches` it learns nothing until the stream ends, so a device that must not lose events across a mid-stream failure needs its own outbound buffer and a resume point; it cannot rely on the ack to tell it where the server got to.

`events_accepted` counts what was **persisted**, not what was received. A count lower than what was sent means the server declined or failed to store the difference, and `error` describes why.

## Notes

- The RPC set deliberately mirrors the publish patterns available over the message-bus transports rather than introducing a request/response idiom of its own. A server implementing OpenPhase ingestion over gRPC and one consuming from NATS see the same messages.
- There is no read side here. `IngestionService` is push-only, device to server; retrieving stored telemetry is an application concern and is outside the protocol.
- `error` is a human-readable string for diagnosis. It carries no status enumeration, so it is intended for operators and logs rather than for a producer to branch on — transport-level failures still surface as gRPC status codes in the usual way.
