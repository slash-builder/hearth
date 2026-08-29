# Hearth / Quickring wire protocol — canonical schema

This directory is the **single source of truth** for the Hearth wire standard
(still packaged `quickring.v1`; see "Naming" below).

```
proto/quickring/v1/quickring.proto   the canonical v1 schema (envelope + fabric)
```

## Canonical home

**This repo — `slash-builder/hearth` — is the canonical home of the schema, as
of 2026-08-22.** It was previously `quickring/quickring.me`
(`proto/quickring/v1/quickring.proto`); it was extracted here in
`quickring.me@53bf891` because a protocol meant to be independently
implementable should not live inside the consumer marketing site's repo.
`quickring.me`'s `main` no longer contains a `proto/` directory at all. Any
doc, agent charter, or script still pointing at `quickring.me` for the schema
is stale.

## Status

Per **ADR-0002** (`slash-builder/fabric-kit/docs/adr/0002-quickring-as-self-contained-open-standard.md`),
the protocol is a self-contained open standard. This `.proto` and the prose spec
at [`../docs/protocol.md`](../docs/protocol.md) together define it: the `.proto`
is authoritative for the wire shape, the prose is authoritative for semantics.
(ADR-0002 was written before the 2026-08-22 move and still names
`quickring/quickring.me` as the standard's home in places; that is a stale
pointer, not a re-decision — this directory is the home.)

## How implementations consume it

No implementation *owns* the schema — each **vendors a pinned copy** and
generates its own bindings:

- **Fabric Kit** (`slash-builder/fabric-kit`) — the reference SDK; Rust via
  `protox` + `prost-build`. Dart/Swift/Kotlin/C bindings follow via
  `flutter_rust_bridge` / `uniffi` / `cbindgen`.
- **message-kit** (`slash-builder/message-kit`) — implements the **base layer**
  (envelope, tag/T-R-D grammar, RError, addressing) for on-device IPC; vendors
  the same pinned copy and generates its Rust (and Go/Dart as needed) bindings.
  It is its own repo. It formerly lived in `devicenix/core`, which no longer
  exists — DeviceNix was retired into BenixOS / SlashBuilder on 2026-08-22 and
  message-kit was extracted as a standalone Kit.
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

## Naming

The protocol's engineering / OSS name is **Hearth**; **Quickring** is the
consumer brand of the commercial layer built on it. The package is still
`quickring.v1` and the WebSocket subprotocol identifier still carries the old
name. Renaming them is a **coordinated wire break** sequenced by the
messaging-architect under **QR-33**, not a unilateral rename in any one repo.

## Encodings

- **protobuf** — canonical (this schema).
- **JSON profile** — a co-equal encoding for platforms without native protobuf,
  derived deterministically from this schema. Tracked as **QR-31**; protobuf
  remains the source of truth.
