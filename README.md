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

~~**Known staleness, not fixed by this move**: `docs/encryption.md`'s own
header is dated 2026-08-16 and describes a **pre-implementation**
state.~~ **Fixed 2026-08-29 (CLUS-54).** `docs/encryption.md`'s status
header, its §1.2 status table, and its §12 "what is NOT implemented"
section now describe reality: the QR-136 programme executed and was
countersigned 2026-08-20, and E0/E1 sealing round-trips are live-verified
end to end, with real device pairing and real chat encryption shipping in
Courier. One honest caveat is recorded there rather than papered over —
`fabric-kit`'s current E0 path is live but not yet byte-conformant to
§7, closed under QR-228 / QR-229.

**Not yet done, tracked separately**: the `.proto` package is still named
`quickring.v1`, not yet renamed to reflect Hearth (the coordinated Hearth
rename pass under QR-33, marked governance-load-bearing 2026-08-22).
Repos that vendor or build against this proto have not all been repointed
at this repo yet — that's a real build-dependency migration, not done
here. `slash-builder/message-kit` **is** repointed (pinned to this repo's
`935712a`, 2026-08-28); `fabric-kit`, `hub`, `gateway`, and `courier` are
not. Docs across those repos that still name `quickring/quickring.me` as
the schema's home were corrected on 2026-08-29 under CLUS-53.

## License

Apache-2.0 on any code; CC-BY-4.0 on the specification documents.
