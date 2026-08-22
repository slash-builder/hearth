# Quickring wire protocol — canonical schema

This directory is the **single source of truth** for the Quickring wire standard.

```
proto/quickring/v1/quickring.proto   the canonical v1 schema (envelope + fabric)
```

## Status

Per **ADR-0002** (`quickring/fabric-kit/docs/adr/0002-quickring-as-self-contained-open-standard.md`),
Quickring is a self-contained open standard. This `.proto` and the prose spec at
[`../docs/protocol.md`](../docs/protocol.md) together define it: the `.proto` is
authoritative for the wire shape, the prose is authoritative for semantics.

## How implementations consume it

No implementation *owns* the schema — each **vendors a pinned copy** and
generates its own bindings:

- **Fabric Kit** (`quickring/fabric-kit`) — the reference SDK; Rust via
  `protox` + `prost-build`. Dart/Swift/Kotlin/C bindings follow via
  `flutter_rust_bridge` / `uniffi` / `cbindgen`.
- **DeviceNix message-kit** (`devicenix/core`) — implements the **base layer**
  (envelope, tag/T-R-D grammar, RError, addressing) for on-device IPC; vendors
  the same pinned copy and generates its Rust (and Go/Dart as needed) bindings.
- **Third-party clients** — vendor the pinned copy, or implement the wire by
  hand (and, for protobuf-less platforms like Roku/BrightScript, the JSON
  profile — **QR-31**).

Pin to a tagged version of this repo; do not track `main` for production builds.

## Two layers, one schema

The Envelope's `oneof` enumerates every v1 message so a packet capture is
self-describing. The base-vs-fabric split (ADR-0002) is an ownership-of-behavior
boundary, not a file boundary:

- **Base layer** — `Envelope`, `tag` correlation, the T/R/D grammar, `RError`,
  addressing. Implemented on-device by message-kit.
- **Fabric layer** — sessions, pub/sub, pairing, presence, catch-up.
  Implemented by the Quickring daemon / hub.

## Versioning

The schema is a permanent, versioned, one-way-door contract. Changes are
additive; field numbers for not-yet-defined messages are reserved; removing or
repurposing a number is a major-version bump. Major version is carried in the
package (`quickring.v1`), the WebSocket subprotocol (`quickring.v1`), and the
endpoint path (`/v1`). See `../docs/protocol.md` § Versioning.

## Encodings

- **protobuf** — canonical (this schema).
- **JSON profile** — a co-equal encoding for platforms without native protobuf,
  derived deterministically from this schema. Tracked as **QR-31**; protobuf
  remains the source of truth.
