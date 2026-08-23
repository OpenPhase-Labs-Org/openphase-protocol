# Contributing to OpenPhase

OpenPhase is an open protocol specification, not an application. Contributions are mostly schema changes — new messages, new fields, vendor-extension code blocks, clarified semantics. This document covers how to propose those changes, the rules they need to follow, and the workflow for getting them landed.

If you want to contribute to a *consumer* of OpenPhase (e.g., the TSIGMA reference ingestion server), see that project's own CONTRIBUTING guide.

---

## Table of contents

- [Code of conduct](#code-of-conduct)
- [Project scope](#project-scope)
- [Schema design rules](#schema-design-rules)
- [The IHR vs NTCIP boundary](#the-ihr-vs-ntcip-boundary)
- [Adding a new field or message](#adding-a-new-field-or-message)
- [Adding a new domain `.proto` file](#adding-a-new-domain-proto-file)
- [Vendor extensions](#vendor-extensions)
- [Versioning and backwards compatibility](#versioning-and-backwards-compatibility)
- [Reference implementation requirements](#reference-implementation-requirements)
- [Documentation](#documentation)
- [Branch and commit workflow](#branch-and-commit-workflow)
- [Pull request checklist](#pull-request-checklist)
- [License](#license)

---

## Code of conduct

Be respectful, be specific, and assume good intent. Hostile, harassing, or discriminatory behavior toward any contributor will result in removal from the project. Disagree with code, not with people.

---

## Project scope

OpenPhase covers the **wire format** for traffic-signal telemetry, control, and analytics. In scope:

- Indiana Hi-Resolution event log (`ihr_events.proto`)
- SAE J2735 SPaT for V2X (`spat.proto`)
- NTCIP 1202 v03A MIB-equivalent objects (`ntcip.proto`)
- Device health, security alerts, fault snapshots, self-learning bit discovery
- Transport contracts (envelopes, batches, `IngestionService` gRPC)

Out of scope:

- Any specific implementation (decoders, storage, dashboards, reports)
- Vendor-private extensions that aren't published under MPL 2.0
- Application-layer policies (auth, rate limiting, retention) — those belong in the consuming application

If you're proposing something that doesn't fit the wire-format scope, it probably belongs in TSIGMA or another consumer.

---

## Schema design rules

These are non-negotiable for anything merged into `openphase.v1`:

- **proto3 only.** No `proto2` syntax.
- **Field numbers are forever.** Once published in a tagged release, a field number cannot be reused or changed. Removing a field is fine; reusing the number is not.
- **Standard types.** Use `google.protobuf.Timestamp` (nanosecond) for time, `string` for free-form identifiers, `uint32` for non-negative bounded integers, `bytes` for OCTET STRINGs and binary blobs.
- **No optional `message` fields without a clear default semantic.** Document what "unset" means.
- **Enums always start with an `_UNSPECIFIED = 0` value.** Required by proto3.
- **Bitmask integers get a parallel bool-field message.** When a spec defines a bitmask (e.g., NTCIP 1202 `phaseOptions`), provide both the raw `uint32` field and a sub-message that unpacks the bits into named bools, so consumers can pick whichever is more ergonomic.
- **OID comments on every NTCIP field.** Every field that maps to a real NTCIP MIB object must have its OID and original camelCase name in a comment, e.g.:
  ```proto
  uint32 walk = 2;  // .1.1.2.1.2  phaseWalk (0-255 SECONDS)
  ```
- **Units in comments.** When a numeric field has a unit (seconds, tenth seconds, minutes, hundredths of a meter), state it in the comment in CAPS where ambiguity has historically caused bugs (e.g., "SECONDS" rather than "tenths").
- **Range constraints in comments.** If the source spec restricts values (e.g., `INTEGER (0..255)`), state the range. proto3 won't enforce it.

---

## The IHR vs NTCIP boundary

OpenPhase deliberately separates two distinct standards:

- **Indiana Hi-Resolution Data Logger Enumerations** (Sturdevant et al., INDOT/Purdue, November 2012) — the de-facto event-log spec used by every ATSPM-compatible controller. Lives in `ihr_events.proto`. The `EventType` enum's integer value equals the IHR event code.
- **NTCIP 1202 v03A** (May 2019) — the OID-addressed MIB protocol for actuated signal controllers. Lives in `ntcip.proto`. NTCIP **does not** define an event-code enumeration; communication is via SNMP GET/SET on OIDs.

Do not conflate the two:
- Don't add an event-code enum to `ntcip.proto`. NTCIP doesn't have one.
- Don't add OID-mapped configuration messages to `ihr_events.proto`. IHR is purely the event log.
- If a contribution spans both (e.g., adding a new event code that also exposes a configuration knob), split it into two PRs against the appropriate file.

---

## Adding a new field or message

1. **Find the source.** Cite the spec section (page number + revision) for any field that mirrors NTCIP/IHR/J2735. Inventing fields without a spec source is grounds for rejection.
2. **Pick the smallest field number that's free.** Re-use existing message slots before creating new ones; consumers benefit from compact varint encoding for low-numbered fields.
3. **Add the OID/spec citation in the comment.**
4. **Update the matching docs page in `docs/`.** Every public message and enum is documented there. If you add a field, add a row.
5. **Ship a test in the reference implementation.** See [Reference implementation requirements](#reference-implementation-requirements).

---

## Adding a new domain `.proto` file

Adding an entirely new schema file is rare and warrants discussion before code. Open an issue first describing:

- The use case
- Why it doesn't fit any existing file
- The expected message shape
- Whether it carries a payload type that should be added to the `IntersectionUpdate.payload` `oneof` in `common.proto`

Once consensus is reached, the new file should:

- Use `package openphase.v1;`
- Include the standard MPL header (copy from any existing proto)
- Have a corresponding documentation file in `docs/<filename>.md`
- Be listed in `README.md` and `docs/OVERVIEW.md`

---

## Vendor extensions

Vendor-specific event codes (typically in the IHR 200-255 range) and proprietary fields are explicitly **not allowed** in the canonical `EventType` enum or any other shared enum. Vendors who need to publish them have two options:

1. **Use the raw escape hatch.** `AtspmEvent` carries `code` (typed enum) — for codes outside the enum, set `code = EVENT_TYPE_UNSPECIFIED` and document the convention out-of-band. (We may add an `extended_code` field for this in a future revision.)
2. **Open an issue proposing standardization.** If multiple vendors are using the same code for the same semantic, that's a candidate for the canonical enum.

Vendors that fork and ship modified `openphase.v1` schemas are violating the MPL 2.0 license — see the [Breaking Change Policy](README.md#breaking-change-policy) in the README.

---

## Versioning and backwards compatibility

The current package is `openphase.v1`. The rules:

- **Additions are non-breaking** and welcome inside `openphase.v1`. New messages, new fields with new numbers, new enum values — all fine.
- **Modifications and removals are breaking** and require a new package (`openphase.v2/`). The previous version remains available indefinitely. Field renumbering, semantic changes to existing fields, removing required fields, and changing enum integer values are all breaking changes.
- **Renaming an identifier without changing its number is non-breaking on the wire** (proto3 uses field numbers, not names) but IS breaking for code that imports the generated stubs by name. Treat identifier renames as breaking.

If you propose a breaking change, also propose the migration story for current consumers.

---

## Reference implementation requirements

Every PR that adds or changes a message, field, or enum value must include a corresponding update to the reference implementation (currently TSIGMA) demonstrating that the schema compiles, generates clean Python stubs, and round-trips through the existing decoder/test harness. Specifically:

- Regenerate the `_pb2.py` (and `_pb2_grpc.py` if you touched a service) and commit them in the consuming repo.
- Add or update a test in `tests/unit/` that exercises the new field via the decoder or service.
- For new IHR event codes, the constant must also be added to the consuming repo's `events.py` SDK with the canonical name.

If you don't have access to the reference implementation, link to a fork in the PR description and a maintainer will run the verification.

---

## Documentation

Every proto file has a matching markdown file in `docs/`. When you change a proto:

- Update the field/enum tables in the matching `docs/*.md`
- If you add a new domain, add `docs/<name>.md` and link it from `docs/OVERVIEW.md` and the top-level `README.md`
- Cite the source spec (NTCIP 1202 §X.Y.Z, J2735 §A.B, IHR Purdue 2012) in both the proto comments and the docs page

---

## Branch and commit workflow

- **Branch off `main`.** Use a descriptive branch name: `feat/ihr-vendor-extensions`, `fix/preempt-units`, `docs/contributing-guide`.
- **One logical change per PR.** Sprawling PRs get sent back.
- **Atomic commits.** Each commit should leave the protos in a compilable state.
- **Commit message format.**
  ```
  <area>: short imperative summary (≤ 70 chars)

  Optional body explaining the why and any spec references. Wrap at
  ~72 chars. Reference issue numbers as #123. For schema changes,
  include the spec section being implemented or clarified.
  ```
  Examples: `ihr_events: add EVENT_PHASE_HOLD_RELEASED (Purdue 2012 §42)`, `ntcip: correct phaseOptions bit 13 (Actuated Rest In Walk)`, `docs: add ihr_events reference table`.

---

## Pull request checklist

Before opening a PR, verify:

- [ ] All `.proto` files compile cleanly with `protoc` (no warnings)
- [ ] Generated `_pb2.py` / `_pb2_grpc.py` updated in the reference implementation
- [ ] Reference-implementation test suite passes
- [ ] Every new field has an OID/spec citation comment with units and range where applicable
- [ ] Matching `docs/*.md` updated
- [ ] If you added a new domain proto, it's listed in `README.md` and `docs/OVERVIEW.md`
- [ ] If your change is breaking under the [versioning policy](#versioning-and-backwards-compatibility), the PR description explains the migration story and the change is targeted at `openphase/v2/`, not `openphase/v1/`
- [ ] PR description explains the *why*, not just the *what*

---

## License

OpenPhase is licensed under the Mozilla Public License 2.0 (MPL-2.0) — see [LICENSE](LICENSE).

By submitting a contribution you agree to:

1. License your contribution under the MPL-2.0.
2. Confirm that you have the right to make the contribution (e.g., your employer's IP policy permits it, or you are the sole author).

There is no separate Contributor License Agreement (CLA) at this time. If we add one before the project's first stable tag, contributors will be notified.

---

## Reporting issues

- **Spec ambiguities or proto bugs:** open an issue with the spec section, the proto file/line, and the observed vs. expected behavior.
- **Implementation interop bugs:** open an issue in the relevant consumer repo (TSIGMA, etc.), not here, unless the proto itself is wrong.
- **Security vulnerabilities in the protocol design:** do NOT open a public issue. Email the OpenPhase Labs security mailbox; we'll acknowledge within 72 hours.
