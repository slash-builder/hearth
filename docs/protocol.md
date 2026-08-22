# Quickring Wire Protocol — v1

**Status:** v0 implemented (2026-06-06, tag `v0`) — the v0 subset (sessions,
pub/sub, presence digest) is live in the hub (`quickring/hub`), Fabric Kit, and
the v0 agent. Pairing, catch-up, `DPresence`, unsubscribe, dedup, rate limits,
and E2E remain spec-only. Canonical schema: [`../proto/quickring/v1/quickring.proto`](../proto/quickring/v1/quickring.proto).
**Audience:** Quickring hub developers, SDK developers, third-party
integrators.
**Scope:** The full Quickring wire standard — both the **base layer** (the
envelope, tags, the T/R/D grammar, errors, addressing) and the **fabric layer**
(sessions, networked pub/sub, pairing, presence, offline catch-up, E2E) — spoken
between Quickring agents and the Quickring hub over WebSocket. The canonical
schema is [`../proto/quickring/v1/quickring.proto`](../proto/quickring/v1/quickring.proto);
this document is normative for semantics. See "Protocol layering" for who
*implements* each layer.

---

## Protocol layering

Quickring's standard is two layers, **both owned by Quickring** and both defined
here (ADR-0002). The split is who *implements* each layer, not who owns it.

| Layer | Covers | Implemented by |
|-------|--------|----------------|
| Base layer | the envelope (`tag` + `oneof`), tag correlation, the T/R/D grammar, the `RError` mechanism, the addressing model | DeviceNix **message-kit** on-device (and any client's transport core) |
| Fabric layer | session lifecycle, networked pub/sub wire, pairing, presence, offline catch-up, E2E encryption, the feed convention | the Quickring daemon / hub |

The canonical Envelope is defined in this standard's `.proto`
([`../proto/quickring/v1/quickring.proto`](../proto/quickring/v1/quickring.proto)) —
**not** imported from elsewhere. DeviceNix message-kit is a *conforming
implementation* that adopts the Quickring envelope as its canonical on-device bus
format: a message is one Quickring envelope end to end, and crossing the box
boundary only swaps the *transport* (`AF_UNIX` socket ↔ WebSocket). The Quickring
daemon registers with the message-kit router as the **remote-transport provider**.

> **Note (ADR-0002, 2026-06-05):** earlier drafts said the envelope was owned by
> the messagekit base protocol and merely *imported* by Quickring. That is now
> reversed — the envelope is part of the Quickring standard, defined here, and
> message-kit implements it. The "The envelope" and "Tag discipline" sections
> below are therefore **canonical**, not historical. QR-14 tracks any further doc
> reorganization; QR-13 the DeviceNix ↔ Quickring integration.

---

## Design ethos

Quickring's wire protocol is heavily influenced by **Plan 9's 9P
protocol** — not in encoding (we use protobuf, not 9P's hand-rolled
binary format), but in *discipline*:

- A small, fixed set of message types. v1 has **24** envelope
  variants (9 T/R pairs, plus `RError` and five D-pushes); aim to
  stay under 30 for the life of the protocol. The discipline is the
  point, not a hard number — every new type should earn its place.
  Six slots of headroom is not much: the next additions should be
  new *steps* of existing interactions (a server-issued handle plus
  a step enum, the 9P `Twalk` pattern) or application-layer
  payloads, not new variants.
- Every client request is paired with exactly one server response,
  correlated by a per-message **tag**.
- Server-pushed messages (deliveries, presence updates) are a
  separate, visibly-distinct category.
- Multi-step interactions use server-issued session handles instead
  of inventing new message types per step.
- Errors are first-class: any request can be answered with an error
  response carrying the same tag.
- The protocol is versioned from message one. Version negotiation
  happens before any other traffic.

The goal: a developer reading a packet capture should be able to tell
instantly which direction a message flows, whether it expects a
reply, and what interaction it's part of.

---

## Transport

Quickring is two services, not one. Don't conflate them:

- **`api.quickring.me`** — a conventional HTTP API (signup, account
  recovery, initial auth-token exchange, device registration).
  Request/response, no long-lived connections. **Rust + axum** per the
  Lockamy Studios language convention locked 2026-05-30 (pivoted from
  Go + Gin); consumes Service Kit (LS-41) for scaffolding. This
  document does **not** cover the HTTP API.
- **`ws.quickring.me`** — the hub: the long-lived WebSocket service
  this document specifies. Implemented in Rust on top of `axum`'s
  WebSocket support (`axum::extract::ws`), which sits on `tokio` +
  `hyper` + `tokio-tungstenite`. The one-reader / one-writer concurrency
  model the hub needs maps directly onto tokio's async task pattern
  (a read task + a write task per connection sharing the WebSocket
  via `SplitSink` / `SplitStream`).

- **Channel:** WebSocket. TLS terminated at Cloudflare; origin uses
  Cloudflare Origin Certificates.
- **Endpoint:** `wss://ws.quickring.me/v1`
- **Subprotocol:** `quickring.v1` (negotiated in the WebSocket
  handshake; future protocol versions get new subprotocol strings).
- **Framing:** One protobuf-encoded `Envelope` per WebSocket binary
  frame. No multi-frame messages in v1.
- **Max message size:** 1 MiB. Larger payloads (file transfer) are
  handled out-of-band via WebRTC or Bitchain in v1.5+ and never
  traverse the hub.
- **Timestamps:** All timestamps on the wire are unix milliseconds,
  UTC. No exceptions, no local time, no timezone offsets. In Go this
  is `time.Now().UTC().UnixMilli()`; in Dart, `DateTime.now()
  .toUtc().millisecondsSinceEpoch`. Display-local conversion happens
  only at the UI edge, never on the wire.

---

## Message categories

Every message belongs to exactly one of three categories, indicated
by the prefix of its type name:

| Prefix | Direction | Semantics |
|---|---|---|
| **T** | Client → Server | A client request. Carries a tag. The server **must** respond with exactly one matching R-message or RError carrying the same tag. |
| **R** | Server → Client | A server response to a previously-received T-message. Carries the same tag as the request. |
| **D** | Server → Client | A server-pushed message, unsolicited. Carries `tag = 0`. The client does not respond. |

This three-way split is the core discipline. When designing a new
interaction, the first question is: "Is this T/R (request/response)
or D (push)?" There is no fourth category.

---

## The envelope

Every message on the wire is a single `Envelope`:

```protobuf
syntax = "proto3";
package quickring.v1;

message Envelope {
  // Correlation tag. Echoed by the server in the matching R-message
  // or RError. Set to 0 for D-messages. Every T-message in v1
  // expects a response and therefore carries a non-zero tag; there
  // are no fire-and-forget T-messages in v1 (TPing included — it
  // always gets an RPing).
  uint32 tag = 1;

  // Exactly one of these is set per envelope.
  oneof body {
    // Session lifecycle
    TVersion     t_version     = 10;
    RVersion     r_version     = 11;
    THello       t_hello       = 12;
    RHello       r_hello       = 13;
    TPing        t_ping        = 14;
    RPing        r_ping        = 15;

    // Topic operations
    TSubscribe   t_subscribe   = 20;
    RSubscribe   r_subscribe   = 21;
    TUnsubscribe t_unsubscribe = 22;
    RUnsubscribe r_unsubscribe = 23;
    TPublish     t_publish     = 24;
    RPublish     r_publish     = 25;

    // Offline catch-up
    TCatchUp     t_catchup     = 30;
    RCatchUp     r_catchup     = 31;

    // Device pairing
    TPair        t_pair        = 40;
    RPair        r_pair        = 41;

    // Roster read
    TRoster      t_roster      = 50;
    RRoster      r_roster      = 51;

    // Error response (replaces any R-message)
    RError       r_error       = 99;

    // Server-pushed deliveries (D-messages)
    DDeliver        d_deliver          = 100;
    DPresence       d_presence         = 101;
    DPairClaim      d_pair_claim       = 102;
    DPairResult     d_pair_result      = 104;
    DSessionRevoked d_session_revoked  = 105;
  }

  reserved 52 to 59, 106;
}
```

Every message type defined anywhere in this document appears in this
`oneof`. There are no "implied" or "to-be-added" variants; if a
message isn't here, it isn't part of v1.

(Field number 103 is intentionally skipped to leave 102–103 grouped
as the pairing D-pushes' immediate neighbourhood while keeping the
slot open for a future `DPair*` push without renumbering.)

Two reservations are enforced by the schema rather than left to a
comment, so `protoc` rejects anything that tries to claim them:

- **52–59** are held for the roster family (see "Roster read").
- **106** is held for a future `DRosterChanged` push, which is
  deliberately **not** defined in v1 — see "What's deliberately not
  in v1". Un-reserving a number is then a deliberate act, which is
  the point.

The `oneof` makes the message type explicit and allows the protocol
to add new types in future versions without breaking older clients
(they ignore unknown variants).

---

## Tag discipline

- Tags are 32-bit unsigned integers, allocated by the client.
- A client must not reuse a tag while a response is still
  outstanding for it.
- Tag `0` is reserved for D-messages (server pushes). Every
  T-message in v1 carries a non-zero tag and expects a response.
- Clients may allocate tags sequentially or randomly; the server
  does not interpret tag values, only echoes them.
- The server tracks outstanding tags per connection. If a client
  sends a T-message with a tag that is already in flight, the server
  responds with `RError(tag=duplicate_tag, code=TAG_IN_USE)`.

**One T, one R.** Every T-message receives exactly one terminal
response — either its matching R-message or an `RError` — carrying
the same tag, after which the tag is free for reuse. A T-message
**never** produces two responses on the same tag. Where an
interaction needs to signal something later (e.g. device-pairing
approval, which waits on a human), that later signal is a
D-message on `tag = 0`, not a second R. This rule is what lets the
SDK model every request as a single promise that resolves or
rejects exactly once.

The client correlates an `RError` to its originating request purely
by tag. `RError` deliberately does **not** carry a field naming
which request type it answers — the SDK already holds the
tag → request-type mapping, and duplicating it on the wire would be
redundant and another thing to keep consistent.

The SDK shields developers from tag allocation entirely; tags are an
implementation detail of the protocol layer.

---

## Session lifecycle

A Quickring session is the lifetime of one authenticated WebSocket
connection. Every session traverses these states in order:

```
   (connected) ──> Version ──> Hello ──> Ready ──> (closed)
```

### 1. Version negotiation

The **first message** on every connection, in both directions, is a
version negotiation. No other traffic is permitted until version is
agreed.

```protobuf
message TVersion {
  // Highest protocol version the client supports.
  // For v1 this is exactly "quickring.v1".
  string version = 1;

  // Maximum envelope size the client is willing to receive, in bytes.
  // Informational; servers may use this to chunk large RCatchUp
  // responses. The hard ceiling is 1 MiB regardless of this value.
  uint32 max_envelope_size = 2;
}

message RVersion {
  // Highest version both client and server support. May be lower
  // than the client's offered version if the server supports an
  // older protocol.
  string version = 1;

  // Server's max envelope size (the actual ceiling for this
  // connection: min of client offer and server limit).
  uint32 max_envelope_size = 2;

  // Server-issued random nonce for device proof-of-possession at
  // THello. A paired client signs (hello_challenge ‖ account_id ‖
  // device_id) with its device key and returns the signature in
  // THello.device_key_signature. Empty from servers that do not
  // challenge; a client that holds no device key ignores it. Adds no
  // round trip — RVersion already precedes THello.
  bytes hello_challenge = 3;

  // Hub build/version string. Diagnostic only — informational, and it
  // MUST NOT drive any protocol behavior (see below). Empty from a hub
  // that does not set it.
  string server_version = 4;
}
```

If the server cannot agree on a version (the client offered a
version the server does not implement), the server responds with
`RError(code=VERSION_INCOMPATIBLE)` and closes the connection.

Version strings follow the form `quickring.v<major>`. Within a
major version, the protocol is backward-compatible: a v1.3 server
accepts v1.0 clients. Breaking changes bump the major version. The
hub supports `N` and `N+1` simultaneously for a minimum of 90 days
during a major version transition.

`RVersion.server_version` is a **diagnostic-only** field: the hub's
own build/version string, exposed so a client can surface which hub
build it is talking to (e.g. a Version/Diagnostics screen). It is the
server-side counterpart to the client's `ClientInfo` in `THello` —
metadata for humans, never for machines. It carries three hard rules:

- **It MUST NOT drive protocol behavior.** Interoperability and
  negotiation are governed **only** by `version` (field 1). A client
  MAY display `server_version` but MUST NOT branch on it, gate a
  feature on it, or parse it for any behavioral decision. If a client
  needs to condition behavior on server capability, that is a
  `version` bump or a negotiated field — never this string.
- **It is unauthenticated.** `server_version` is not covered by any
  signed or authenticated data (unlike `hello_challenge`), so a
  client MUST NOT make any trust or security decision from it.
- **It is empty-tolerant.** A hub that predates this field, or that
  chooses not to advertise a build, sends it empty; a client renders
  an empty value as "unknown" and MUST NOT treat it as malformed.

The value is an opaque display string. The RECOMMENDED form is
`"<semver>+<shortsha>"` (e.g. `CARGO_PKG_VERSION` plus an optional
`GIT_SHA`), and it SHOULD stay short (≤ 64 bytes); clients MUST NOT
depend on any particular format.

### 2. Hello (authentication)

Once version is agreed, the client authenticates and identifies its
device.

```protobuf
message THello {
  // Account-authentication credential. Modeled as a oneof from day
  // one so that adding client-certificate auth in a later version is
  // an additive change, not a breaking one. v1 implements ONLY the
  // bearer_token arm; the others are reserved and rejected with
  // RError(code=AUTH_INVALID) if sent.
  oneof auth {
    // Bearer token issued by the HTTP auth flow at api.quickring.me.
    string bearer_token = 1;
    // Reserved for v2+. Not implemented in v1.
    // bytes client_cert = 5;
  }

  // The device's stable identifier (UUID). Issued at first device
  // registration; persisted in local storage on the agent.
  string device_id = 2;

  // Optional resume token from a previous session on this device.
  // Presenting a valid resume_token lets the server re-establish
  // the session's prior subscriptions without a fresh round of
  // TSubscribe calls. It does NOT itself replay missed messages —
  // to recover messages sent while offline, the client issues
  // TCatchUp per subscription after RHello. If absent or invalid,
  // the client re-subscribes from scratch.
  optional bytes resume_token = 3;

  // Client metadata for diagnostics. Not used for any auth decision.
  ClientInfo client_info = 4;

  // Device proof-of-possession binding. NOT a oneof arm — it is
  // orthogonal to `auth` and coexists with bearer_token (field 6 is
  // the number the oneof historically reserved for device-key auth).
  // When present, an Ed25519 signature over
  // (hello_challenge ‖ account_id ‖ device_id) using the device's
  // registered key. The `auth` credential still resolves the account;
  // if this signature is present the server ADDITIONALLY verifies it
  // against the registered key for (account, device_id) and rejects
  // with RError(code=AUTH_INVALID) on mismatch — binding a
  // self-asserted device_id to a key the holder can prove, without
  // replacing token auth. Absent, or no key on file for the device →
  // unchanged bearer-only behavior (a device that has never paired is
  // not rejected). A standalone field precisely so it can bind
  // alongside a token now and later stand alone (tokenless
  // proof-of-possession: auth unset + signature present, once the
  // server can resolve an account from the key) without a wire break.
  bytes device_key_signature = 6;
}

message ClientInfo {
  string platform   = 1;  // "macos", "ios", "android", "web", "linux-headless", "thunderhead"
  string sdk_name   = 2;  // "quickring-dart", "quickring-go", "quickring-js", or third-party identifier
  string sdk_version = 3; // semver
  string app_name   = 4;  // optional; for third-party integrations
}

message RHello {
  // Session identifier. Useful for log correlation; not security-sensitive.
  string session_id = 1;

  // Account this device belongs to.
  string account_id = 2;

  // Resume token to present in the next THello to resume from this
  // point. Rotated each session.
  bytes resume_token = 3;

  // Snapshot of other devices on this account and their current
  // presence. Lets the client populate its UI immediately without
  // needing to wait for DPresence pushes.
  repeated DevicePresence presence_digest = 4;

  // Server time at the moment of RHello, in unix milliseconds.
  // Clients use this to align local clocks for sort order and TTLs.
  int64 server_time_ms = 5;
}

message DevicePresence {
  string device_id = 1;
  PresenceState state = 2;
  int64 last_seen_ms = 3;
  ClientInfo client_info = 4;
}

enum PresenceState {
  PRESENCE_UNKNOWN = 0;
  PRESENCE_OFFLINE = 1;
  PRESENCE_ONLINE  = 2;
  PRESENCE_AWAY    = 3;
}
```

Auth failures (`RError(code=AUTH_INVALID)`, `RError(code=AUTH_EXPIRED)`)
close the connection. The client should re-acquire an auth token via
the HTTP API and reconnect.

### 3. Ready

After `RHello`, the connection is ready for normal traffic. The
client may now issue subscribes, publishes, catch-ups, and pairing
requests. The server may now push `DDeliver` and `DPresence`.

### Heartbeats

```protobuf
message TPing {
  // Optional client-side timestamp for round-trip latency measurement.
  int64 client_time_ms = 1;
}

message RPing {
  int64 server_time_ms = 1;
  int64 client_time_ms = 2;  // Echoed from request, if provided
}
```

Clients send `TPing` every 30 seconds while idle. The server uses
the heartbeat to refresh the device's presence TTL in Redis. A
connection with no traffic (including pings) for 90 seconds is
considered abandoned and closed by the server.

Every `TPing` also re-validates the connection's registration against
current state (has this device's registration been replaced or
removed? has the account owner revoked it?). If it no longer checks
out, the server closes the connection instead of answering with
`RPing` — no `RError` either, since the tag simply never resolves and
the client is expected to treat a closed connection the same way it
treats any other drop. This bounds a revoked-but-live session to at
most one heartbeat interval (~30s) even with **no wire message at
all**, which is why it exists independently of, not instead of,
`DSessionRevoked` below: the two together are an immediate path and a
bounded fallback, not two implementations of the same thing.

---

## Topic operations

### Topic naming convention

Topics are slash-delimited strings, case-sensitive, ASCII only.
Components:

| Prefix | Shape | Purpose |
|---|---|---|
| `account/<account_id>/device/<device_id>` | Targeted | Deliver to one specific device on an account. |
| `account/<account_id>/broadcast` | Fanout | Deliver to all online devices on an account ("carbons"). |
| `account/<account_id>/feed` | Feed | The account's feed (posts + comments, as envelope-typed payloads). |
| `dm/<a>/<b>` | Direct message | DMs between two accounts. The two account ids are sorted ASCII-ascending and joined with `/`. |
| `team/<team_id>/...` | Team | **Reserved for v2.5.** Do not use in v1. |

Topic names are validated at subscribe / publish time. Invalid names
return `RError(code=TOPIC_INVALID)`.

Authorization for each topic is enforced server-side:

- A device may subscribe to any topic under its own `account/<account_id>/...`.
- A device may subscribe to a `dm/<a>/<b>` topic if its account is
  one of `a` or `b`.
- A device may publish to `account/<own_id>/...` and to
  `dm/<own_id>/<other>`.
- Cross-account publishing requires that the recipient has accepted
  a contact relationship (mechanism out of scope for this doc;
  enforced at the HTTP API layer).

### Subscribe

```protobuf
message TSubscribe {
  string topic = 1;

  // Field 2 (since_ms) is RESERVED and MUST NOT be re-used. A
  // subscription is ALWAYS forward-only from the moment it is
  // registered. Recovering messages published before the subscription
  // is TCatchUp's job and TCatchUp's alone (QR-134 ruling, 2026-08-18):
  // there is exactly one catch-up path, not two. See "Catch-up" below
  // for the canonical reconnect recipe.
  reserved 2;
  reserved "since_ms";
}

message RSubscribe {
  // Server-issued handle for this subscription. Echoed in DDeliver
  // messages so the client knows which subscription a delivery
  // belongs to.
  string subscription_id = 1;

  // Resolved topic (in case the server canonicalised something).
  string topic = 2;

  // Server time at subscribe. The client anchors the next since_ms it
  // sends on a TCatchUp for this topic to this value (or to the
  // server_time_ms of the last delivery it actually processed).
  int64 server_time_ms = 3;
}
```

**A subscription is forward-only.** `TSubscribe` begins live delivery from the
instant the hub registers it; it never replays history. This is a deliberate
single-path decision (QR-134): catch-up across a gap is `TCatchUp`, which alone
carries the `limit`/`has_more` pagination and flow control a resuming mobile
client needs. A half-designed `since_ms` on `TSubscribe` would be a second,
unbounded, un-paginated catch-up path racing live delivery — protocol
minimalism forbids two ways to do one thing. The field was declared but silently
ignored by the hub; rather than wire up the inferior path it is removed and its
number reserved.

**Canonical reconnect recipe (normative).** To resume a topic across a
disconnect without a gap and without unbounded replay:

1. `TSubscribe(topic)` — resubscribe first, so live delivery covers everything
   from *now* forward with no gap.
2. `TCatchUp(subscription_id, since_ms = <server_time_ms of the last delivery
   processed before the drop>)`, paging on `has_more` until drained, to recover
   the backlog between the last-seen point and the resubscribe instant.
3. **Deduplicate on the hub-assigned `message_id`** (the ULID), which is stable
   across both paths. The window between last-seen and resubscribe may be
   delivered by *both* catch-up and live delivery; `message_id` dedup makes the
   overlap harmless. This is why client- and hub-side dedup (see `client_message_id`
   under Publish) is load-bearing, not optional — subscribe-then-catch-up is only
   correct because duplicates collapse on a stable id.

A successful `TSubscribe` returns a `subscription_id` that the
server uses in subsequent `DDeliver` messages. Multiple subscriptions
to the same topic are allowed and return distinct `subscription_id`s
(useful for SDK consumers that wire two independent handlers).

### Unsubscribe

```protobuf
message TUnsubscribe {
  string subscription_id = 1;
}

message RUnsubscribe {
  // Empty on success.
}
```

After `RUnsubscribe`, the server stops pushing `DDeliver` messages
for that subscription_id. Any DDeliver already in flight may still
arrive at the client; the client should ignore deliveries for
unknown subscription_ids.

### Publish

```protobuf
message TPublish {
  string topic = 1;

  // Opaque payload. The hub does not interpret payload bytes.
  // For E2E-encrypted topics this is ciphertext; the cleartext
  // structure is defined by the application layer (e.g. feed
  // posts, DMs, presence-rich-state, file-transfer signaling).
  bytes payload = 2;

  // Optional client-assigned message id for deduplication on
  // retry. If omitted, the server assigns one. Deduplication is
  // scoped to (account_id, client_message_id) and held in Redis
  // with a 10-minute TTL — NOT scoped to the connection. This is
  // deliberate: the case a client_message_id exists to handle is
  // "connection dropped mid-publish, client doesn't know if it
  // landed, reconnects on a new connection, retries." A
  // connection-scoped dedup would miss exactly that case. If a
  // TPublish arrives whose (account_id, client_message_id) the hub
  // has seen within the TTL window, the hub returns the original
  // RPublish (same message_id, same server_time_ms) instead of
  // publishing again. Outside the TTL window, a repeat is treated
  // as a new publish.
  optional string client_message_id = 3;

  // Optional payload type hint for client-side routing. Not
  // interpreted by the hub. Convention:
  //   "post" — feed post
  //   "comment" — feed comment
  //   "dm" — direct message
  //   "presence_rich" — extended presence state
  //   "signal" — WebRTC signaling for file transfer (v1.5+)
  //   "bitchain_ref" — Bitchain manifest reference, payload contains
  //                    a bitchain:// URI or serialised manifest pointer
  //                    (v1.5+; see file transfer note below)
  optional string payload_type = 4;
}

message RPublish {
  // Server-assigned message identifier: a ULID generated by the
  // hub. ULID gives time-ordered, globally-unique ids with no
  // cross-instance coordination (unlike a per-topic counter, which
  // would be a coordination point across hub instances behind
  // Redis). Globally unique, not merely per-topic.
  string message_id = 1;

  // Server-assigned timestamp (unix ms UTC). Redundant with the
  // ULID's embedded time component for ordering, but retained as
  // an explicit field so clients never have to parse a ULID to
  // sort. The ULID is the tiebreaker when two messages share a
  // millisecond.
  int64 server_time_ms = 2;
}
```

The hub does not read payload contents. All payload interpretation
happens at the application layer in agents and SDKs. This is the
load-bearing decision that makes the hub an opaque relay; see the
project brief at `agents/projects/quickring.md` for the locked E2E
stance.

> **`client_message_id` idempotency is part of the wire contract, and it is
> enforced, not advisory (QR-118).** The de-duplication behaviour above — a
> `TPublish` whose `(account_id, client_message_id)` the hub has seen within the
> 10-minute window returns the *original* `RPublish` (same `message_id`, same
> `server_time_ms`) rather than publishing again — is a protocol guarantee
> clients build retry logic on, not an optional optimisation. It MUST be
> implemented before any E2E (E1) topic goes live: once payloads are ciphertext,
> each retry carries a *fresh nonce and therefore distinct ciphertext bytes*, so
> no subscriber and no catch-up consumer can collapse a duplicate by payload
> comparison — `(account_id, client_message_id)` becomes the only surviving
> dedup key in the system. **Normative concurrency requirement:** the guarantee
> MUST hold for *concurrent* retries, not only sequential ones. The hub MUST
> claim `(account_id, client_message_id)` **atomically before** assigning a
> `message_id` or relaying (e.g. Redis `SET NX`); the claim's winner publishes
> and records its `RPublish` under the key, and every loser reads back that
> stored `RPublish`. Publishing first and recording the dedup key afterward
> reopens the duplicate under a two-client race — the same first-write-ordering
> failure class as ADR-0004's device-0 provisioning, and it must be tested with
> genuinely concurrent publishes, not a sequential happy path.

---

## Server-pushed messages

### DDeliver

The server pushes `DDeliver` whenever a message lands on a topic
the client is subscribed to.

```protobuf
message DDeliver {
  // Which subscription this delivery satisfies.
  string subscription_id = 1;

  // The topic the message was published to (may be redundant with
  // the subscription, but explicit for SDK debugging).
  string topic = 2;

  // Server-assigned message id (matches the RPublish from the sender).
  string message_id = 3;

  // Sender's device id. Useful for filtering "my own" messages
  // on broadcast topics where the sender also receives the delivery.
  string sender_device_id = 4;

  // Sender's account id.
  string sender_account_id = 5;

  // Server timestamp the message was published (sort key within topic).
  int64 server_time_ms = 6;

  // The opaque payload bytes from the publisher.
  bytes payload = 7;

  // Echoed payload_type hint from publisher, if any.
  optional string payload_type = 8;
}
```

Tag is always `0`. Clients do not respond to DDeliver.

### DPresence

The server pushes `DPresence` whenever the presence state of a
device the client is "watching" changes. In v1, every client
automatically watches presence for all other devices on its own
account; cross-account presence subscriptions are a v2 feature.

```protobuf
message DPresence {
  string device_id = 1;
  string account_id = 2;
  PresenceState state = 3;
  int64 last_seen_ms = 4;

  // Only populated when state transitions to ONLINE.
  optional ClientInfo client_info = 5;
}
```

Tag is always `0`.

### DSessionRevoked

The server pushes `DSessionRevoked` to a device whose session has
been administratively invalidated — its registration was replaced
(a key rotation, e.g. device-0 recovery) or an account owner revoked
it. Sent **immediately before** the server closes that device's
connection, so a client that receives it can tell a deliberate
revocation apart from a transient network drop and respond
accordingly (surface it to the user; do not silently reconnect with
the same credentials).

```protobuf
message DSessionRevoked {
  string device_id = 1;
  string account_id = 2;
  SessionRevokeReason reason = 3;
  int64 revoked_at_ms = 4;
}

enum SessionRevokeReason {
  SESSION_REVOKE_REASON_UNKNOWN = 0;
  REVOKED_BY_OWNER = 1;  // an account owner explicitly revoked this device
  DEVICE_REMOVED = 2;    // the device's registration was replaced or removed
  SESSION_EXPIRED = 3;   // reserved; no active producer in v1
}
```

Tag is always `0`. `reason` is deliberately a closed enum, not a
free-text field — nothing about *why* beyond the enum value belongs
on this push (redaction discipline: this push crosses the same
hub-plaintext relay as everything else in v1).

A device is not guaranteed to receive this push — it may be offline
at the moment of revocation, or the push itself may be dropped. The
`TPing` re-validation described under "Heartbeats" above is the
bound for that case: at most one heartbeat interval (~30s) rather
than an unbounded window. `DSessionRevoked` is the immediate path;
the `TPing` check is the fallback that does not depend on delivery.

See `dlockamy/vault: wiki/decisions/quickring-capability-model-fuchsia.md`
§6.2 and `wiki/decisions/quickring-protocol-security-roadmap.md` §5 for
why revocation needs both a push and a heartbeat-bounded fallback:
distributed revocation has no equivalent of a kernel-mediated handle
close, so delivery of the push can never be assumed.

---

## Offline catch-up

```protobuf
message TCatchUp {
  // The subscription to catch up. Implies the topic and authorization.
  string subscription_id = 1;

  // Receive messages with server_time_ms > since_ms.
  int64 since_ms = 2;

  // Maximum messages to return in this batch. The server may return
  // fewer. If the catch-up set exceeds this limit, RCatchUp signals
  // has_more = true and the client should issue a follow-up TCatchUp
  // with since_ms set to the last returned message's server_time_ms.
  uint32 limit = 3;
}

message RCatchUp {
  // Replayed deliveries. Same shape as DDeliver. The client should
  // process these identically to live deliveries — the SDK is
  // expected to unify the two paths.
  repeated DDeliver deliveries = 1;

  // True if there are more messages beyond this batch.
  bool has_more = 2;

  // Server time at the moment of RCatchUp. Use this (not the last
  // delivery's server_time_ms) as the next since_ms to avoid race
  // conditions with concurrent publishes.
  int64 server_time_ms = 3;
}
```

The server's short-TTL message store (DynamoDB) retains messages
for **7 days** in v1. `TCatchUp` with `since_ms` older than 7 days
returns `RError(code=CATCHUP_HORIZON_EXCEEDED)` and the client
should treat its local state as authoritative going forward.

---

## Device pairing

Pairing is the multi-step interaction that adds a new device to an
existing account. It's the "scan a QR code, done" experience from
the brief, and the most complex flow in v1. Modeled after 9P's
`Twalk`: one message type with multiple step kinds, carrying a
server-issued session id across steps.

### First-account bootstrap is out of scope

Pairing adds a device to an account that **already has** at least one
trusted device. How the *first* device on a brand-new account comes
to exist is deliberately **not** part of this protocol — there is no
device to pair from, so the wire stays silent on it and keeps the
standard vendor-neutral. An implementation is free to bootstrap the
first device however it likes (e.g. self-issuing account keys
locally). The first device then looks like any other account device
to the wire, and all subsequent devices join through ordinary
pairing.

Quickring (the hosted product) bootstraps via an operator-held
**"device 0"** provisioned at web/API signup: device 0 holds the
account's first key and plays the existing-device role in the
ordinary pairing flow for the user's first real device, and doubles
as a user-controllable recovery anchor. That is a deployment choice
above the protocol, recorded in **ADR-0004**; the wire below is
unchanged.

### The flow

```
Existing device                   Hub                   New device
───────────────                   ───                   ──────────
TPair(step=INITIATE) ──────────────►
                  ◄──────────── RPair(pair_session_id, qr_payload)
[displays QR code]

                                                       [scans QR;
                                                        extracts
                                                        pair_session_id
                                                        and bootstrap
                                                        endpoint]

                                                       (opens WebSocket)
                                                       TVersion / RVersion
                                                       (no THello — paired
                                                        identity is being
                                                        established here)

                                  ◄──────────────────  TPair(step=CLAIM,
                                                              pair_session_id,
                                                              new_device_pubkey,
                                                              new_device_sealing_pubkey,
                                                              sealing_binding_sig,
                                                              proposed_device_info)
                                  ──────────────────►  RPair(step=CLAIMED)
                                                       [terminal response to
                                                        the CLAIM tag; new
                                                        device now waits for
                                                        a DPairResult push]
                  ◄──────────── DPairClaim
                                  (notify existing
                                   device of claim;
                                   carries both keys
                                   and the reserved
                                   new_device_id)
[displays "Pair
 this device?"
 prompt with
 fingerprint over
 BOTH of the new
 device's keys]

TPair(step=APPROVE,
      pair_session_id,
      attestation_signature,
      signed_at_ms) ──────────────►
                  ◄──────────── RPair(step=COMPLETE)
                                  (terminal response to
                                   the APPROVE tag)
                                  ──────────────────►  DPairResult(approved,
                                                              account_credentials,
                                                              device_id)
                                                       [persists creds,
                                                        opens a fresh
                                                        connection, sends
                                                        THello]
```

The key discipline point: the new device's `TPair(CLAIM)` gets an
**immediate** terminal `RPair(CLAIMED)` — it does not block waiting
for a human to approve on the other device. Approval (or rejection,
or timeout) arrives later as a `DPairResult` server-push on
`tag = 0`. This keeps the "one T, one R" rule intact: no request is
ever left hanging on human latency, and no tag ever receives two
responses. `DPairClaim` (to the existing device) and `DPairResult`
(to the new device) are both D-messages, defined below.

### TPair / RPair shape

```protobuf
message TPair {
  PairStep step = 1;
  oneof step_body {
    TPairInitiate initiate = 2;
    TPairClaim    claim    = 3;
    TPairApprove  approve  = 4;
  }
}

enum PairStep {
  PAIR_STEP_UNKNOWN  = 0;
  PAIR_STEP_INITIATE = 1;  // From existing device: start a pairing
  PAIR_STEP_CLAIM    = 2;  // From new device: claim the pair session via the QR payload
  PAIR_STEP_APPROVE  = 3;  // From existing device: approve the claim
}

message TPairInitiate {
  // Optional human-readable label for the pairing session, shown
  // in logs ("Pairing initiated from MacBook Pro").
  string label = 1;
}

message TPairClaim {
  // The pair_session_id encoded in the QR code.
  string pair_session_id = 1;

  // The new device's freshly-generated Ed25519 identity public key
  // (32 bytes). Used by the existing device to display a fingerprint
  // for confirmation.
  bytes new_device_pubkey = 2;

  // Self-reported device info.
  ClientInfo client_info = 3;

  // Proposed human-readable device name ("Doug's iPad").
  string proposed_device_name = 4;

  // --- F7 additions (encryption.md §5.3) ---

  // The new device's X25519 *sealing* public key (32 bytes, RFC 7748
  // encoding). This is the key a content key or a sealed secret is
  // later addressed to. It is carried here, in the claim, for one
  // reason: so that the approver's attestation signature covers it.
  // A sealing key that arrives after the attestation has no path back
  // to the human tap, and the only thing asserting it would then be
  // the hub — which is the substitution attack the attestation exists
  // to prevent, at the layer where substitution buys plaintext.
  bytes new_device_sealing_pubkey = 5;

  // The device's own Ed25519 signature binding the sealing key above
  // to its identity key (encryption.md §3.2, context string
  // "qr-sealing-binding-v1"). 64 bytes.
  bytes new_device_sealing_binding_sig = 6;

  // The `created_at_ms` that is INSIDE the binding signature's signed
  // material. Carried explicitly because a signature over a timestamp
  // nobody transmits can never be verified by anyone.
  int64 sealing_created_at_ms = 7;
}

message TPairApprove {
  string pair_session_id = 1;

  // Ed25519 signature by an existing-device signing key, proving the
  // user on the existing device approved. What it signs depends on
  // the attestation version (below):
  //   v1 — the bare 32-byte new_device_pubkey, no domain separation.
  //   v2 — the domain-separated material of encryption.md §5.3.
  bytes signature = 2;

  // F7. The approver's own signing timestamp, which is *inside* the v2
  // signed material and therefore cannot be supplied by the hub after
  // the fact. Ignored for a v1 approval.
  int64 signed_at_ms = 3;
}

message RPair {
  PairStep step = 1;

  oneof step_body {
    RPairInitiated initiated = 2;  // → existing device, response to INITIATE
    RPairClaimed   claimed   = 3;  // → new device, response to CLAIM
    RPairComplete  complete  = 4;  // → existing device, response to APPROVE
  }
}

message RPairInitiated {
  // Server-issued session id; encoded into the QR shown by the
  // existing device. Short-lived (60 seconds).
  string pair_session_id = 1;

  // QR payload to display. Includes: pair_session_id, the WebSocket
  // bootstrap endpoint, and a short-lived single-use bootstrap
  // credential. Encoded as a URL: quickring://pair?...
  string qr_payload = 2;

  // Server-side TTL for the pair session, in unix ms.
  int64 expires_at_ms = 3;
}

message RPairClaimed {
  // Terminal response to the new device's TPair(CLAIM). Acknowledges
  // the claim was registered; it does NOT carry credentials. The
  // actual approval happens out of band on the existing device and
  // arrives later as a DPairResult push (see below). This split is
  // what keeps the "one T, one R" rule intact — the CLAIM tag is
  // resolved immediately, not held open across human latency.
  string pair_session_id = 1;

  // When the pair session expires if no approval arrives. After
  // this, the new device should expect a DPairResult with
  // outcome = OUTCOME_TIMEOUT (or simply give up).
  int64 expires_at_ms = 2;
}

message RPairComplete {
  // Terminal response to the existing device's TPair(APPROVE).
  // Confirms the hub accepted the approval and has dispatched (or
  // will dispatch) credentials to the new device via DPairResult.
  // The pair session is now closed.
  string new_device_id = 1;
  string new_device_name = 2;
}

message DPairClaim {
  // Pushed to existing device(s) on the account when a new device
  // claims a pair session. Lets the existing device prompt the
  // user with the new device's fingerprint and proposed name.
  // Tag is always 0.
  string pair_session_id = 1;
  bytes new_device_pubkey = 2;
  string proposed_device_name = 3;
  ClientInfo new_device_client_info = 4;

  // --- F7 additions (encryption.md §5.3) ---

  // Relayed verbatim from TPairClaim. The hub neither mints nor
  // rewrites these; it is a relay for them, exactly as it is for
  // new_device_pubkey. Empty (zero-length / zero) when the claiming
  // device predates F7 — that is a v1 pairing.
  bytes new_device_sealing_pubkey = 5;
  bytes new_device_sealing_binding_sig = 6;
  int64 sealing_created_at_ms = 7;

  // F7. The device id the hub has RESERVED for this claim.
  //
  // Minted at CLAIM rather than at APPROVE, which is a change from v1's
  // ordering, because `new_device_id` is inside the v2 attestation
  // material the approver signs — and the approver signs before the hub
  // would otherwise have minted it. A reserved id is not a registered
  // device: nothing is written to the device roster until APPROVE
  // succeeds, and an unapproved session is deleted with its reservation.
  // Empty on a v1 pairing.
  string new_device_id = 8;
}

message DPairResult {
  // Pushed to the NEW device to report the final outcome of a
  // pairing it claimed. This is the asynchronous counterpart to
  // RPairClaimed: the new device received an immediate RPairClaimed
  // for its CLAIM tag, then waits for this push. Tag is always 0.
  string pair_session_id = 1;

  PairOutcome outcome = 2;

  // Populated only when outcome == OUTCOME_APPROVED. These are the
  // credentials the new device persists and uses for its first
  // THello on a fresh connection.
  optional PairCredentials credentials = 3;
}

enum PairOutcome {
  OUTCOME_UNKNOWN  = 0;
  OUTCOME_APPROVED = 1;  // Existing device approved; credentials present
  OUTCOME_REJECTED = 2;  // Existing device declined the pairing
  OUTCOME_TIMEOUT  = 3;  // Pair session expired before approval
}

message PairCredentials {
  string device_id = 1;
  string account_id = 2;
  string bearer_token = 3;  // For the new device's first THello
  bytes  resume_token = 4;
}
```

### Pairing security notes

- The QR payload contains a bootstrap credential that allows
  *only* a `TPair(CLAIM)` against the named pair_session_id. It is
  not a general auth token.
- The pair session expires after 60 seconds. If the new device
  doesn't claim in time, the existing device must restart pairing.
- Approval requires a signature from an existing-device key, not
  just a UI tap. This prevents a hub compromise from completing
  pairings without user consent on a trusted device.
- The new device's public keys are the trust anchor for E2E message
  encryption. The fingerprint shown on the existing device during
  APPROVE is the user's last line of defence against a hub-mediated
  attacker, and under F7 it is computed over **both** keys — see
  "The fingerprint" below.

### The pairing attestation — v1 and v2

The approver's signature is not thrown away after it is checked; it is
persisted as a **pairing attestation**, the record that device X
approved device Y. A chain of these, terminating at the household
authority root, is what lets a client learn a peer's public key
without taking the hub's word for it.

There are exactly two attestation versions, and an implementation
**MUST** be able to tell them apart without inferring it from which
fields happen to be present.

**v1 — identity key only.** The signed material is the bare 32-byte
`new_device_pubkey`, with no context string and no length prefix. It
covers the Ed25519 identity key and nothing else. **A v1 attestation
means no sealing-key attestation exists** — not a weaker one, none.
It cannot be upgraded: the approver signed material that did not
contain the sealing key and cannot be asked to re-sign the past.

**v2 — identity key and sealing key, domain-separated.** The signed
material is:

```
attestation_sig = Ed25519_sign(
    approver_identity_sk,
      "qr-pair-attest-v2"                  // 17-byte ASCII context
    ‖ 0x00                                 // domain separator terminator
    ‖ len16(new_device_id) ‖ new_device_id_utf8
    ‖ new_device_ed25519_pubkey            // 32 bytes
    ‖ new_device_x25519_pubkey             // 32 bytes
    ‖ be64(signed_at_ms)
)
```

`len16` is a big-endian `uint16` byte length; `be64` is a big-endian
64-bit two's-complement encoding of the signed `int64`. Every
variable-length field is length-prefixed, and the context string plus
its `0x00` terminator are mandatory. They are not decoration: v1
signs a bare 32-byte Ed25519 public key, and once a *second* 32-byte
signature material exists in the system (the sealing key), an
undomain-separated signature produced for one purpose can be presented
as a signature for the other.

**Which version applies is a property of the pair session, never a
claim made by a message.** If the CLAIM carried a sealing key, the
approval is v2 and is verified as v2; if it did not, the approval is
v1. There is deliberately no "attestation version" field on
`TPairApprove` — a client-asserted version would let a claim with no
sealing key be approved as though it had one.

A hub that stripped the sealing key out of `DPairClaim` would cause the
approver to produce a v1 attestation. That is a **denial**, not a
downgrade: the resulting device has no sealing-key attestation, so
under the fail-closed rules below nothing is ever wrapped to it. This
is the intended behaviour, and it is why stripping buys an attacker
nothing.

**Fail closed. Normatively:**

- A roster entry that fails any check MUST be treated as **absent**,
  not as degraded. No wrap is produced for it, no seal is addressed to
  it, no content key reaches it.
- Failure MUST NOT fall back to trusting the hub-served value.
- Failure MUST NOT fall back to publishing in plaintext.
- An unknown attestation version, an unknown context string, or an
  unrecognized key encoding MUST deny. Never "try anyway."
- A device with no attestation at all MUST be treated as unverifiable
  and therefore not wrappable.

### The fingerprint

Shown on the existing device during APPROVE, and on the new device's
own screen, for the human to compare. Under F7 it covers both keys,
so that the thing the human compares is the thing the signature
covers:

```
fingerprint = SHA-256(
      "qr-device-fingerprint-v1" ‖ 0x00
    ‖ ed25519_pubkey ‖ x25519_pubkey
)  →  first 20 bytes, RFC 4648 base32 (uppercase, unpadded),
      rendered in groups of 4 characters
```

20 bytes is 32 base32 characters, rendered as 8 groups of 4.

A device with no sealing key cannot have a v2 fingerprint computed for
it — there is nothing to hash in the second position, and substituting
zeros would make two different devices display the same string. Such a
device is a v1 pairing and is displayed with the legacy identity-only
rendering; implementations MUST NOT pad, zero-fill, or otherwise
manufacture a sealing key to force a v2 fingerprint.

### Conformance vector

Every binding MUST reproduce these bytes exactly. This vector is the
conformance contract for the v2 attestation material and the
fingerprint; a binding that produces different bytes is broken,
regardless of whether it interoperates with any particular hub build.

```
new_device_id            = "0191d2c8-0000-7000-8000-000000000001"  (36 bytes)
new_device_ed25519_pubkey = 32 bytes, 0x01 repeated
new_device_x25519_pubkey  = 32 bytes, 0x02 repeated
signed_at_ms              = 1755300000000

v2 attestation material (SHA-256 of the material bytes):
  SHA-256 = 0b622adde23708d5569aa4cd78e1b2ef85c19a61f9c5af3baa1596261c50f602
  (the material itself is 128 bytes)

fingerprint over the two keys above:
  JMCQ VBQD JJHJ HDZI NA37 OVDU 6SW2 T4NV
```

The E2E key management story is detailed in a sibling document,
`docs/encryption.md` — normative for key management, content
confidentiality, and the sealed-payload primitive. F7, described
above, is the wire half of that document's §5.3.

---

## Roster read

Pairing produces attestations. The **roster read** is how one device
gets to see another's — the wire path that delivers one device's
public keys to another so that a content key can be wrapped to it.
It is the wire half of `encryption.md` §5.4, which stays normative
for the trust rules; this section is normative for the wire shape.

A device learns a peer's public keys by asking the hub for the
peer's roster entry and then **verifying that entry against the
household authority root it already holds**. The hub is a
**directory**, never an **authority**: it stores and returns records
it cannot forge.

`TRoster` is a request and `RRoster` is its single terminal
response, exactly like every other T/R pair — one T, one R, same
tag. There is no roster push in v1 (see "What's deliberately not in
v1").

### TRoster / RRoster

```protobuf
message TRoster {
  // Device ids to fetch. EMPTY MEANS THE WHOLE ROSTER for this
  // connection's account.
  repeated string device_ids = 1;
}

message RRoster {
  // One entry per device the hub could answer for. A requested
  // device id with no entry is ABSENT, not an error.
  repeated RosterEntry entries = 1;

  // The household authority root every attestation chain in this
  // reply must terminate at, and — where the household has been
  // through the device-0 handoff — the transition attestation that
  // authorizes it.
  bytes household_authority_root_pubkey = 2;
  optional TransitionAttestation authority_transition = 3;

  // True only when the WHOLE-ROSTER form was cut short. Never true
  // for a request that named device_ids explicitly.
  bool truncated = 4;

  int64 server_time_ms = 5;
}

message RosterEntry {
  string device_id = 1;

  // Ed25519 identity key. Always present.
  bytes ed25519_pubkey = 2;

  // X25519 sealing key and its binding. All three present, or all
  // three absent.
  bytes x25519_pubkey         = 3;
  bytes sealing_binding_sig   = 4;
  int64 sealing_created_at_ms = 5;

  // The F6/F7 pairing attestation, verbatim as the approver signed
  // it.
  optional PairingAttestation attestation = 6;
}

message PairingAttestation {
  uint32 version            = 1;  // 1 or 2; unknown DENIES
  string approver_device_id = 2;
  bytes  approver_pubkey    = 3;  // as it stood AT SIGNING TIME
  bytes  signature          = 4;
  int64  signed_at_ms       = 5;
}

message TransitionAttestation {
  string approver_device_id              = 1;  // "device-0" today
  bytes  approver_pubkey                 = 2;  // the OLD root
  bytes  household_authority_root_pubkey = 3;  // the NEW root
  bytes  signature                       = 4;
  int64  signed_at_ms                    = 5;
}
```

**There is deliberately no `account_id` on `TRoster`.** The account
is the session's, established at `THello` and proved against
`RVersion.hello_challenge`. Accepting one from the request would be
an authorization decision taken from client-supplied data, which is
the shape of every cross-tenant read bug ever written. The same rule
runs in the other direction: `TransitionAttestation` carries no
`account_id` either, even though the account id is *inside* the
material its signature covers — the verifier reconstructs it from
the session (`RHello.account_id`), never from the reply, so the hub
cannot choose the domain its signature is checked in.

`TransitionAttestation.approver_device_id` is **not** covered by
that signature. It is a diagnostic label, never an authenticated
assertion.

`PairingAttestation.version` is a `uint32` rather than an enum on
purpose. The rule is *deny on unknown*, not *map unknown onto a
default* — and an unknown enum value decodes to zero in most
bindings, losing exactly the number a denial needs to log.

### The attestation does not repeat the keys it signs

`PairingAttestation` carries the signature and the signer. It does
**not** carry its own copy of `new_device_pubkey` or
`new_device_x25519_pubkey`. The verifier reconstructs the v2 signed
material (see "The pairing attestation — v1 and v2" above) from the
**entry's own** `device_id`, `ed25519_pubkey` and `x25519_pubkey`.

This is the load-bearing property of the whole design: **a hub that
substitutes a key into a `RosterEntry` necessarily breaks the
signature it is serving in the same message.** No consistent lie is
available to it. If the attestation carried its own copy of the
keys, a hub could serve a matched (attested-key, substituted-key)
pair and leave the verifier to notice a discrepancy it may not have
been written to check. **Do not add those fields back for
"completeness."** If a review ever sees the attestation carrying its
own copy of the keys, that is the bug.

### What the hub can and cannot do in this role

**It CAN:** withhold an entry or the whole roster; serve a stale
entry or roll back to an earlier one; delay, reorder, or refuse; and
observe *which device asked about which device* — the wrap graph,
already a known metadata leak (`encryption.md` §8.2).

**It CANNOT:** substitute a key; fabricate a device (there is no
approver signature chaining to this household's root); transplant an
attestation from another household (its approver chains to *that*
household's root); upgrade a v1 attestation to v2 (different signed
material, and the approver cannot be asked to re-sign the past); or
make a stripped sealing key look present (all-three-or-none, and a
v1 attestation is a **denial** rather than a downgrade).

**Withholding and staleness are denial, not substitution.** That is
the honest boundary of this design, and it is the boundary of every
key directory without a transparency log. Forcing a directory to
*prove* freshness is the key-transparency problem, which is out of
scope for v1 and is named here rather than papered over.

### Normative rules

**Fetch before wrap.**

> A roster entry **MUST NOT** be taken from any cache when it is
> being used to produce a wrap or a seal. The `TRoster` → verify →
> wrap sequence is one act.

Caching an entry for display, for a device list, for a fingerprint
screen, or as an intermediate in a chain being verified is fine.
Caching it to decide where a content key goes is forbidden. This is
stated instead of a staleness TTL because every such number is
arbitrary and every arbitrary number is eventually raised by whoever
is debugging a slow test. The cost is one round trip at the moment a
human is already tapping Approve, once per device, ever.

It also closes the one security-relevant staleness case: a device
removed from the account yesterday must not receive today's content
key because someone's cache still lists it. A cooperative hub
answers correctly; a hostile hub can only withhold, which fails
closed.

**Truncation is not absence.**

> `truncated = true` **MUST NOT** be used to conclude anything about
> a device that is not present in the reply.

A wrap decision **MUST** rest on either a reply to a request that
named `device_ids` explicitly, or a whole-roster reply with
`truncated = false`. "I asked for the roster, that device wasn't in
it, therefore it isn't paired" is exactly the reasoning a truncated
reply invalidates, and exactly the reasoning that drops a device out
of a household's key set.

**Fetch the whole roster when you are about to wrap (SHOULD).** A
`TRoster(device_ids = ["B"])` discloses to the hub that A is about
to seal something to B. Requesting the whole roster hides the
target. If the reply comes back `truncated`, the client falls back
to the explicit form and accepts the disclosure — stated as a trade
rather than hidden, though at realistic household sizes truncation
does not fire.

A partial answer is a success, not a failure: a requested device id
with no entry is simply absent from `entries`, and the request still
returns `RRoster`, never an `RError`.

### What a reader MUST verify, in order

Failing closed at each step. This list is reproduced from
`encryption.md` §5.4.5, which remains normative for it.

1. `household_authority_root_pubkey` equals the root this device
   already holds. If it does not, and `authority_transition` is
   present, verify the transition attestation under the *old* root
   and accept the new root only through it. A root that arrives with
   no chain to a root already held **MUST** deny; a new root is
   never accepted on the hub's word.
2. For each entry, `attestation` is present. Absent → the device is
   unverifiable → treat as absent.
3. `attestation.version` is a version this implementation supports.
   Unknown → deny. Never trial-verify against each known material in
   turn.
4. `attestation.approver_pubkey` belongs to a device whose own chain
   terminates at the household authority root, accepting the
   transition where the chain crosses the handoff. Unterminated →
   deny.
5. `attestation.signature` verifies under `approver_pubkey` over the
   material for its version, **reconstructed from this entry's own
   `device_id`, `ed25519_pubkey` and `x25519_pubkey`**. Mismatch →
   deny.
6. If a sealing key is to be used: all three sealing fields are
   present, and `sealing_binding_sig` verifies under this entry's
   `ed25519_pubkey` over the `encryption.md` §3.2 material including
   `sealing_created_at_ms`. Missing or failing → the entry has no
   usable sealing key → no wrap.
7. For a **v1** attestation, step 6 is not sufficient: a v1
   attestation does not cover the sealing key, so the binding proves
   only that the device asserted the key to itself, not that a human
   approved it. A v1-attested device is **not wrappable**.

Any failure: the entry is discarded, the peer is treated as absent,
and the failure is surfaced. It is never downgraded to a warning and
never retried against a relaxed check.

### Size, capping, and old peers

A fully populated entry is ~320 bytes on the wire (36-byte device id
+ 32 + 32 + 64 + 8 + ~144 attestation + protobuf framing). 100
devices is ~32 KB, against 64 KiB payload guidance and the 1 MiB
envelope ceiling — **size is ruled out as a consideration.** The hub
**SHOULD** cap the whole-roster form so `RRoster` stays inside the
connection's negotiated `max_envelope_size`; **256 entries** is the
recommended cap and is where `truncated` comes from.

Backward behaviour needs no negotiation. A client speaking to a hub
that predates this receives `RError(code=UNKNOWN_MESSAGE_TYPE)` on
its tag — a clean, terminal, one-T-one-R failure it can act on. A
hub speaking to an old client simply never receives a `TRoster`.
There is no capability flag, no feature negotiation, and no
`RVersion` change.

### Why not somewhere else

The alternatives were considered and rejected; the reasoning is in
`encryption.md` §5.4.6 and is summarised here because it is the
question a reader asks first.

- **Extending `presence_digest`** is a category error.
  `DevicePresence` is volatile and TTL-backed; the roster is
  permanent. Merging them forces a join between two lifetimes on
  every presence event and re-ships every device's key material on
  every presence *flap*. Worse, it turns a presence bug into a
  key-distribution bug, in a record that arrives unsolicited where a
  client has no natural place to hang verification.
  **`DevicePresence` and `DPresence` are unchanged by this.**
- **A new field on `RHello`** is a one-shot at the wrong moment.
  `RHello` fires once, at connect; a device that pairs *after* your
  `RHello` is invisible until you reconnect — and the minute after a
  device pairs is exactly when its key is needed.
- **A roster topic with retained messages** only works if
  subscribing yields current *state* rather than *history*, and the
  topic model has no retained-message semantic. Adding one is a
  change to every topic in the protocol.
- **A payload request answered by the hub** would make the hub a
  payload interpreter, creating a second class of "payloads the hub
  does read" — and that class grows. The general rule, which this
  section establishes: **the hub answers in the envelope; devices
  answer in the payload.** If the responder is the fabric itself,
  the request is a T with an R. If the responder is another
  household device, it is a payload on a topic. The seam is the
  responder, not the convenience.
- **An HTTP endpoint** would be a second protocol, a second auth
  path, a second thing to version, and a self-host parity hazard by
  construction.

### Self-host

`TRoster` is answered by whatever hub the household is connected to.
A self-hosted hub answers it identically out of its own storage,
with no call to anything Lockamy operates, and verification is
entirely local against the household authority root. Key
distribution does not depend on any particular hub being reachable,
and a self-hosted household gets bit-identical security properties.

---

## Error responses

Any T-message may be answered with `RError` instead of its matching
R-message. The tag echoes the request's tag.

```protobuf
message RError {
  // Stable error code. Add new codes by appending; never reuse a
  // retired code's number.
  ErrorCode code = 1;

  // Human-readable description. Not for programmatic use; intended
  // for logs and developer debugging.
  string message = 2;

  // Optional structured details. Schema varies by code.
  bytes details = 3;
}

enum ErrorCode {
  ERROR_UNKNOWN              = 0;

  // Session lifecycle
  VERSION_INCOMPATIBLE       = 1;
  AUTH_INVALID               = 2;
  AUTH_EXPIRED               = 3;
  HELLO_REQUIRED             = 4;  // T-message sent before THello
  ALREADY_HELLOED            = 5;  // Second THello on the same connection

  // Protocol
  TAG_IN_USE                 = 10;
  MALFORMED_ENVELOPE         = 11;
  UNKNOWN_MESSAGE_TYPE       = 12;
  ENVELOPE_TOO_LARGE         = 13;

  // Topics
  TOPIC_INVALID              = 20;
  TOPIC_UNAUTHORIZED         = 21;
  SUBSCRIPTION_NOT_FOUND     = 22;
  PUBLISH_RATE_LIMITED       = 23;
  SUBSCRIBE_RATE_LIMITED     = 24;

  // Catch-up
  CATCHUP_HORIZON_EXCEEDED   = 30;

  // Pairing
  PAIR_SESSION_NOT_FOUND     = 40;
  PAIR_SESSION_EXPIRED       = 41;
  PAIR_SIGNATURE_INVALID     = 42;

  // Roster
  ROSTER_RATE_LIMITED        = 50;

  // Generic
  INTERNAL                   = 90;  // Hub-side bug; report to ops
  TEMPORARILY_UNAVAILABLE    = 91;  // Backoff and retry
  ABUSE_DETECTED             = 92;  // Suspicious pattern; connection closing

  reserved 51 to 59;                // held for the roster family
}
```

`RError` never replaces a D-message. D-messages are unsolicited;
errors flow only in response to T-messages.

The roster family deliberately has **no** `ROSTER_DEVICE_NOT_FOUND`.
A requested device id the hub cannot answer for is simply **absent**
from `RRoster.entries`; a partial answer is a normal success and
never fails the request. `ROSTER_RATE_LIMITED` is the only roster
code in v1, and 51–59 are reserved beside it.

---

## Rate limits and abuse

v1 enforces per-connection and per-account limits. Hitting a limit
returns `RError(code=PUBLISH_RATE_LIMITED)` or
`RError(code=SUBSCRIBE_RATE_LIMITED)`; suspicious patterns return
`RError(code=ABUSE_DETECTED)` and close the connection.

Initial v1 limits (subject to public-beta tuning):

| Limit | Value | Scope |
|---|---|---|
| TPublish | 10/sec, 600/hour | Per connection |
| TPublish | 50/sec, 6000/hour | Per account (all devices) |
| TSubscribe | 50 active | Per connection |
| TSubscribe | 200 active | Per account |
| Payload size | 64 KiB | Per TPublish (larger payloads go out-of-band via WebRTC or Bitchain, v1.5+) |
| Connection rate | 5/minute | Per account from new IPs |
| Pair sessions | 3 concurrent | Per account |
| TRoster | not yet set | Per connection and per account |

`ROSTER_RATE_LIMITED` exists as a code so the limit can be turned on
without a wire change; the **numbers** are a public-beta abuse-policy
decision and are deliberately not fixed here. Note when setting them
that fetch-before-wrap makes one `TRoster` per device admitted an
expected, human-paced event, not a hot path — a limit tuned as though
it were a hot path would break the normal flow.

Abuse heuristics live server-side and look only at metadata
(message frequency, recipient graph, connection patterns). The hub
**does not** inspect payload contents, in keeping with the E2E
stance. See `agents/projects/quickring.md` for the locked rationale.

---

## Versioning

- Major version is part of the subprotocol string
  (`quickring.v1`, `quickring.v2`, etc.) and the WebSocket
  endpoint path (`/v1`, `/v2`).
- Within a major version, the protocol is backward-compatible.
  New message types are additive; new fields on existing messages
  must be optional.
- Breaking changes require a major version bump. The hub supports
  the previous major version for at least **90 days** after a new
  major version goes live, to give SDKs and third-party agents
  time to migrate.
- This v1 spec is pre-public-beta. Until v1.0.0 ships publicly,
  the protocol may break without notice. Once public beta opens,
  all changes follow the rules above.

---

## What's deliberately not in v1

These are planned for later phases per the project roadmap. They
do not appear in this version of the protocol.

- **File transfer** (`v1.5`). Two complementary mechanisms, both
  keep file bytes off the hub:
  1. **P2P via WebRTC** — for ad-hoc transfers between two online
     agents. The hub carries signaling envelopes via the existing
     publish/subscribe mechanism (`payload_type = "signal"`) but
     never the file bytes.
  2. **Bitchain manifest references** — for content-addressed
     transfers, especially when the sender may go offline before
     the receiver picks up, or when the same content is fanned out
     to many devices. A `TPublish` carries `payload_type =
     "bitchain_ref"` and a payload containing the manifest URI
     (`bitchain://manifest/<hash>`). The receiving agent resolves
     the manifest via Bitchain's own retrieval path (local cache,
     S3, a Refraction instance, or peer Quickring agents acting as
     Bitchain stores). The hub sees only the reference, never the
     blocks. Natural fit for firmware drops to Thunderhead boxes,
     shared media collections, and any content the user already
     has in their Bitchain store.

  Choice between the two is an application-layer decision: WebRTC
  for interactive "send this now" flows, Bitchain refs for
  durable / fanout / store-and-forward shapes. The wire protocol
  treats both uniformly as opaque payloads with a type hint.
- **`DRosterChanged`** — a push notifying a device that the account
  roster changed. **Envelope field 106 is reserved for it and it is
  deliberately not defined in v1.** It would buy freshness that a
  hostile hub cannot be forced to provide, and against a cooperative
  hub the fetch-before-wrap rule already covers every case that
  matters — so building it now would be a third envelope variant
  delivering a property the design does not rest on. The reservation
  keeps the addition additive and unhurried if publish-cadence data
  ever justifies it.
- **A `roster_version` / `roster_digest` cache hint on `RHello`.**
  Rejected as premature rather than wrong. Its only consumer is a
  cache heuristic, a hostile hub can lie about it freely, and adding
  a field to the one message every implementation must parse
  forever, to save an occasional small round trip, is the wrong
  trade today. It is an additive field later if real telemetry
  justifies it.
- **Roster freshness proofs / key transparency.** The hub can
  withhold or stale a roster entry, and v1 accepts that as a named
  residual (see "Roster read"). Forcing a directory to *prove*
  freshness is the key-transparency problem and is not attempted
  here.
- **Teams / multi-account topics** (`v2.5`). The `team/` topic
  prefix is reserved but unused.
- **Federation** (`v3+`). v1 is hub-anchored. No `@user@otherhub`
  identity, no inter-hub message routing.
- **Server-side search, moderation, or content inspection**. Out
  of scope at the protocol level; locked decision in the project
  brief.

---

## Open questions

Captured for resolution before v0 implementation begins.

**Resolved during the Developer review (kept here for the record):**

- ~~**Auth methods in `THello`.**~~ **Resolved:** `THello.auth` is
  a `oneof`, reserved from day one so additional methods are
  additive later, but v1 implements only the `bearer_token` arm.
  Gets the wire-compatibility safety without the v1 scope.
- ~~**Binding `device_id` to a proven device key at connect.**~~
  **Resolved (additive):** `device_id` was self-asserted at `THello`
  — any holder of an account token could claim any `device_id` on the
  account. `RVersion` now carries a server-issued `hello_challenge`
  and `THello` carries a standalone `device_key_signature` (field 6);
  when present, the hub verifies an Ed25519 signature over
  `(hello_challenge ‖ account_id ‖ device_id)` against the device's
  registered key and rejects `AUTH_INVALID` on mismatch. It is a
  *binding* on top of token auth, not a replacement, and a *standalone
  field rather than a oneof arm* so it coexists with `bearer_token`
  today and can stand alone later (tokenless proof-of-possession, once
  the hub can resolve an account from a key) without a wire break.
  This is the ordered prerequisite for any per-device authorization
  that trusts `sender_device_id`.
- ~~**Exposing the hub build over the wire.**~~ **Resolved (additive,
  2026-08-20):** `RVersion` now carries `server_version` (field 4), the
  hub's own build string, for diagnostic surfaces (Courier's
  Version/Diagnostics screen). It rides the existing handshake with no
  new envelope variant and no extra round trip — the server-side
  counterpart to the client's `ClientInfo`. Ruled diagnostic-only:
  `version` (field 1) alone governs negotiation, clients MUST NOT branch
  on `server_version`, it is unauthenticated, and it is empty-tolerant
  (rendered "unknown", never malformed). Chosen over a dedicated
  diagnostic message on protocol-minimalism grounds — a single
  informational string does not earn a new variant.
- ~~**`payload_type` enum vs. free-form string.**~~ **Resolved:**
  free-form string. `payload_type` is an application-layer hint the
  hub never interprets, and the value space is owned by apps, not
  the protocol — so locking it into the wire proto would force a
  protocol release every time an app invents a message kind. The
  Bitchain-ref addition is the proof case.
- ~~**Pairing: two responses to one tag.**~~ **Resolved:**
  `TPair(CLAIM)` now gets an immediate terminal `RPairClaimed`, and
  approval/rejection/timeout arrives as a separate `DPairResult`
  push. No tag ever receives two responses.

**Still open:**

1. **Should `TPublish` have a `delivery_receipt_requested` flag**
   that asks the hub to push a `DDelivered` (separate from
   `DDeliver`) when each recipient device acknowledges? Current
   draft: no, keep the protocol simple; receipts are an
   application-layer concern.
2. **Should `subscription_id`s be globally unique or
   per-connection?** Current draft: per-connection (server-assigned
   strings, opaque to client, scope = this connection).
3. **Heartbeat cadence** — 30 seconds chosen somewhat arbitrarily.
   Should be load-tested before public beta to balance presence
   accuracy against hub CPU cost.
4. **`DPairResult` delivery when the new device is briefly
   disconnected.** The new device holds an open connection while
   waiting for approval, so in the common case `DPairResult` pushes
   straight down it. But if that connection drops between
   `RPairClaimed` and approval, the push has nowhere to go. Options:
   the new device polls a `TPair(step=POLL)` on reconnect, or the
   pairing bootstrap connection supports the normal resume
   mechanism. Needs a decision before pairing is implemented (v1,
   not v0), but doesn't affect the v0 plaintext demo.

---

## References

- **9P protocol** (Plan 9 from Bell Labs) — the foundational
  influence. http://man.cat-v.org/plan_9/5/intro
- **XMPP** — wisdom borrowed without adopting the protocol:
  - XEP-0198 Stream Management (reconnect / resume / ack semantics)
  - XEP-0280 Message Carbons (multi-device fanout)
  - XEP-0313 MAM (offline catch-up shape)
  - XEP-0384 OMEMO (E2E key management; feeds the encryption.md spec)
- **Project brief:** `agents/projects/quickring.md` in
  `lockamy-studios/agents`. Locked decisions, mandate, phasing.
- **Bitchain:** content-addressed binary versioning. Manifest
  references travel as Quickring payloads; blocks resolve via
  Bitchain's own retrieval path. Repo: `dlockamy/bitchain`.
- **Lockamy Studios developer agent:** `agents/developer.md` —
  stack conventions, error-handling discipline.

---

*Last updated: 2026-06-06 (v0 shipped — tag `v0`). Maintained by the
Messaging Architect agent. Developer review pass complete; the canonical
`.proto` is published at `../proto/quickring/v1/quickring.proto` and the v0
subset is implemented end to end (hub QR-3..6, Fabric Kit QR-7, agent
QR-8/9 — epic QR-1 closed). Ownership reframed per ADR-0002: this standard
owns both layers; message-kit and Fabric Kit are conforming implementations.*
