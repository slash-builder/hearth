# Hearth

The open protocol for distributed personal resources. 9P-inspired — Plan 9's
addressing model, reapplied to a household instead of a data center.
Capability-grants instead of accounts; a thin wire contract instead of a
platform.

Stewarded by [SlashBuilder](https://slashbuilder.com), Lockamy Studios' open-source
organization, as a peer project alongside [BenixOS](https://github.com/slash-builder/core).
[Quickring](https://quickring.me) is the commercial layer built on top —
Hearth itself sells nothing.

## What's here

- **`docs/protocol.md`** — the wire protocol specification: transport,
  framing, session lifecycle, device pairing.
- **`docs/encryption.md`** — key management and content-confidentiality
  design (sealed payloads, the capability-grant model).
- **`proto/quickring/v1/quickring.proto`** — the canonical Protobuf schema.

## Extracted from quickring.me, 2026-08-22

This content previously lived inside `quickring/quickring.me`'s `docs/` and
`proto/` directories — the consumer marketing site's own repo, not a
neutral home for a protocol meant to be independently implementable. Moved
here as part of the SlashBuilder/BenixOS restructuring (see
`dlockamy/context/cross-repo-context.md`'s Corporate structure section).

**Not a history-preserving extraction** — this is a fresh copy of the
content as of `quickring/quickring.me@c043871`, not a git-history import.
For the file history before this point, see that commit and its ancestors
in `quickring/quickring.me`.

**Not yet done, tracked separately**: the `.proto` package is still named
`quickring.v1`, not yet renamed to reflect Hearth (the coordinated Hearth
rename pass under QR-33, marked governance-load-bearing 2026-08-22).
Repointing at this repo is largely done: message-kit's vendored schema
(`proto/VENDORED.md`) and the hub's pin a copy of this repo's
`proto/quickring/v1/quickring.proto`, fabric-kit takes its generated types
from message-kit, and the gateway and Courier vendor no schema of their own
(they go through fabric-kit). Some stale `quickring/quickring.me` references
remain in comments and docs (for example fabric-kit's `Cargo.toml`), and the
`quickring.v1` package-name rename is the part still open.

## Encryption: implementation status

Checked against the reference implementations' source on 2026-10-27. Those
reference implementations (hub, Fabric Kit, the apps) are not public yet, so
readers cannot check these claims against the code themselves.
`docs/encryption.md` is both the design and, in §1.2, a status table. Its
header was corrected on 2026-09-05 (it had described a state where nothing
was implemented), but only some rows were re-audited then, and the code has
moved since. Where `docs/encryption.md` and the code disagree, the code is
what runs; this section is the current summary. Parts of
`docs/encryption.md` known to be behind the code: §1.2 rows 5.4 and 5.6, the
bootstrap description in §4, and §12.

Implemented in the reference client SDK (Fabric Kit) and the reference hub,
and exercised by their integration tests against a live hub and Redis:

- A per-device sealing key, minted on the device at pairing claim, bound to
  the device's identity key, recorded by the hub, and covered by the pairing
  attestation.
- Roster read (`TRoster`/`RRoster`) with client-side verification of the
  attestation chain. A roster entry that fails verification is treated as
  absent.
- E0 as used today: sealing a household content key to each device on the
  roster (`ContentKeyGrant`, `docs/encryption.md` §5.7). The standalone
  `SealedBlob` for arbitrary payloads (§7.2) is **not** implemented.
- E1: payload encryption under a household content key
  (`FabricClient::publish_encrypted`, `Subscription::decrypt_if_sealed`).
- The key-admission exchange between two devices (§5.6).

In the reference app (Courier), as read from source (its tests were not run
for this check):

- The chat send path encrypts each message under E1 and returns an error
  instead of sending when no content key is present. Other publishes, such as
  device-control messages, are not E1-sealed.
- The household content key is provisioned by hand through a beta tool
  (generate, or paste a key). The app does not run the key-admission
  exchange.
- The content key, the identity seed, and the sealing seed are stored in
  ordinary app preferences, not in platform secure storage, which
  `docs/encryption.md` §3.5 requires.
- The household authority-root handoff (§4) exists in the SDK and the
  service, but no app code calls it.

Not implemented: per-resource keys (E2); key rotation and rekey-on-revoke
(E3; the epoch field is carried in the formats and the SDK has no rotation
logic); forward secrecy and key transparency (E4).

What this means for a reader:

- The hub still sees message metadata (topic, sender, timing, size);
  encryption does not change that (`docs/encryption.md` §2.3).
- Frames published before a household has a content key are plaintext to the
  hub (`docs/encryption.md` §11.2).
- The public-claim gates in `docs/encryption.md` §2.4 for custody claims
  (for example "your household key never leaves your device", "we cannot
  access your household") are not met by the shipped app today. Nothing in
  this repository makes those claims.

## License

Apache-2.0 on any code; CC-BY-4.0 on the specification documents.
