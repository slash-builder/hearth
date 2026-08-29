# Hearth Encryption — key management and content confidentiality

**Status:** **SPECIFIED; E0 and E1 ARE IMPLEMENTED AND LIVE.** The QR-136
encryption programme executed and was countersigned on 2026-08-20: X25519
device sealing keys, RFC 9180 HPKE (`DHKEM(X25519, HKDF-SHA256)` /
HKDF-SHA256 / ChaCha20-Poly1305) key wrapping, and XChaCha20-Poly1305 content
encryption are all real, shipping code in `fabric-kit`, with E0 seal → unwrap
and E1 `publish_encrypted` → hub relay → `decrypt_if_sealed` both live-verified
end to end between independent connections. Real device pairing and real chat
encryption ship in Courier (v0.1.33+). Ciphertext is on the wire.

Two honest qualifications:

- **Implemented is not the same as conformant.** `fabric-kit`'s current E0 path
  is live but not yet fully conformant to this document's byte-level spec; that
  gap is being closed under **QR-228 / QR-229**. Where this document and the
  current code disagree, **this document is normative** and the code is the
  thing that moves.
- **E2 and beyond are still design only.** Everything below the cut line
  (per-resource keys, rekey-on-revoke, key epochs, forward secrecy) is
  specified and unbuilt. §1.2's table is the per-section source of truth.

> **Doc-lag note (CLUS-54, 2026-08-29).** Until this edit, this header read
> "**SPECIFIED. NOTHING IN THIS DOCUMENT IS IMPLEMENTED**" and described the
> pre-implementation state as of 2026-08-16 — no X25519 key, no AEAD, no HPKE,
> no ciphertext on the wire. That was overtaken by QR-136 four days later and
> was carried forward unchanged through the 2026-08-22 extraction of this spec
> from `quickring/quickring.me` into `slash-builder/hearth` (which flagged it
> in the repo README as known, unfixed staleness). It was a documentation lag,
> never a statement about the code. The old header's last sentence — "the hub
> reads every payload in plaintext" — is the part that has most concretely
> changed: an E1-encrypted publish reaches the hub as ciphertext and the hub
> routes it on metadata alone. Payloads sent on paths that do not yet encrypt
> still transit in plaintext; §1.2 says which is which, and §11 is the honest
> account of the remaining gap.

**Audience:** Hearth implementers, SDK developers, third-party implementors of
the protocol.

**Scope:** This document is normative for key management, content
confidentiality, and the sealed-payload primitive. It is the sibling document
referenced by [`protocol.md`](protocol.md) ("Pairing security notes"). Wire
semantics remain `protocol.md`'s; algorithms, key lifecycles, byte layouts, and
trust rules are this document's.

**Relationship to `protocol.md`:** where the two disagree about a wire shape,
`protocol.md` wins. Where they disagree about what a key is, who may hold it, or
what a verifier must check, this document wins. `protocol.md` deliberately
contains no algorithm choices; every one of them lives here.

---

## 1. Scope and status

### 1.1 What this document is normative for

- The two household keys, their generation, and the absolute prohibition on
  deriving one from the other (§3).
- The device sealing key: when it is minted, what binds it to a device, and how
  a peer obtains it (§3.2, §5).
- The E1 content-encryption ciphertext format, byte for byte (§6).
- The E0 sealed-payload format, byte for byte (§7).
- The content-key delegation exchange — the protocol act by which a device
  joins the set that can decrypt (§5.6).
- What a verifier MUST check, and that every failure is closed (§§5, 6, 7).
- Rollout and migration, including the honest description of the gap (§11).

It is **not** normative for: the record shape of anything described here (that
is the data-architect's), the wire fields that carry it (messaging-architect's),
or the capability token format (see [the capability
model](#8-capability-tokens-and-encryption--the-seam) and `protocol.md`).

### 1.2 Implementation status

Copying `protocol.md`'s discipline: **nothing here may read as shipped before it
ships** — and, equally, nothing that *has* shipped may keep reading as unbuilt.
This table is the single place to check.

**Last reconciled: 2026-08-29 (CLUS-54), against the QR-136 countersign of
2026-08-20.** The rows below were written on 2026-08-16 and went stale four days
later; the E0/E1 rows are corrected here. A row that says IMPLEMENTED means the
behavior is live, **not** that it is byte-conformant to this document — see the
conformance column note and QR-228/229.

| § | Subject | Programme | Status |
|---|---|---|---|
| 2 | Threat model | — | **SPECIFIED** |
| 3.1 | Device identity key (Ed25519) | pre-E0 | **IMPLEMENTED** (`gateway/identity.rs`, `fabric-kit/pairing.rs`) |
| 3.2 | Device sealing key (X25519) | E0 | **IMPLEMENTED.** Minted and used by `fabric-kit` (`sealing.rs`); bound to the device by the F7 attestation (§5.3) and returned in v2 roster entries with a usable sealing key (QR-137, live-verified 2026-08-20). The 2026-08-16 note that "no client or gateway mints one" is obsolete. |
| 3.3–3.5 | Content keys, `key_id`, never-derive rule | E1 | **IMPLEMENTED.** `fabric-kit/content_key.rs`; a household content key exists and is wrapped per device. The never-derive rule (§3.3) is a MUST and is honored — identity and sealing keys are independent. |
| 3.6 | Key epochs | E3 | **SPECIFIED, DEFERRED.** Field reserved, always `0` at E1. |
| 4 | Root of authority vs. root of custody | E1 | **SPECIFIED.** Bootstrap rework in progress; current code (`api/src/device0.rs`) implements the superseded model. |
| 5.2 | Pairing attestation (F6) | E0 | **IMPLEMENTED.** Persisted by `hub/pairing.rs` (commit `8c63496`, extended by F7 below). |
| 5.3 | F7 — attestation covers the sealing key | E0 | **IMPLEMENTED.** `TPairClaim`/`DPairClaim`/`TPairApprove` fields are in `protocol.md`. Hub-side v2 material and version discrimination are on `hub` `main` (PR #13, merged 2026-08-17); client-side wiring live-verified with real device-0-approved pairing (`fabric-kit` `97b522d`). |
| 5.4 | Roster read (D3) | E0 | **IMPLEMENTED.** `TRoster`/`RRoster`, envelope 50/51 — ruled by messaging-architect 2026-08-17; the 7-step `fetch_roster` verify path is live and returns v2 entries with usable sealing keys (QR-137, `fabric-kit` PRs #10/#12, `72af798`). |
| 5.6 | Content-key delegation exchange | E1 | **IMPLEMENTED, device→device.** The hub-mediated `/keys/*` variant was deleted by ruling (`hub` PR #19, −2607 LOC); the device→device exchange described here is the normative and the shipped one. |
| 6 | Payload encryption | E1 | **IMPLEMENTED.** XChaCha20-Poly1305 under the household content key; `publish_encrypted` → hub relay → `decrypt_if_sealed` live-verified between two independent connections (`fabric-kit` PRs #13/#14, `0d0a287`/`8b0b58e`) and shipping as real chat encryption in Courier v0.1.33+. Payloads published on paths that do not yet call the encrypting API still transit in plaintext — see §11. |
| 7 | Sealed payloads | E0 | **IMPLEMENTED, conformance gap open.** Real RFC 9180 HPKE `mode_base`, DHKEM(X25519, HKDF-SHA256) / HKDF-SHA256 / ChaCha20-Poly1305 (`fabric-kit/seal.rs`, `sealing.rs`); `seal_to_devices` → paired-device unwrap recovers the key byte-identically (`72af798`). **`fabric-kit`'s current E0 path is not yet fully conformant to §7's byte layout; the gap is being closed under QR-228 / QR-229. This document is normative — the code moves, not the spec.** |
| 8 | Capability seam | — | **SPECIFIED.** Capability core not implemented. |
| 9 | Revocation and rekey | E3 | **SPECIFIED, DEFERRED to 2028.** |
| 10 | Self-host decoupling | — | **SPECIFIED and verified against this design.** |
| 11 | Rollout and migration | E1 | **PARTIALLY EXECUTED.** E0/E1 rolled out; §11's migration narrative predates that and still describes the rollout prospectively. One open product call (§11.5). |

**The programme labels** (E0–E4) are the confidentiality decomposition:

| | | |
|---|---|---|
| **E0** | seal a blob to exactly one device | §7 |
| **E1** | content encryption under a single household content key | §6 |
| — | **the cut line** — everything below is gated on a product trigger, not a date | |
| **E2** | per-resource keys | §3.4 |
| **E3** | rekey-on-revoke, key epochs | §9 |
| **E4** | forward secrecy, key transparency, E2E account recovery | out of scope here |

### 1.3 Deviation from the filed outline

This document follows the twelve-section outline filed in the security roadmap,
with one addition: **§11 (Rollout and migration) is new**, and the outline's
former §11 and §12 become §12 and §13. Migration was not in the outline and is
load-bearing — an already-paired household does not acquire a content key by
wishing.

---

## 2. Threat model

Everything in this document answers to this section. A control that does not
address an adversary listed here is decoration; a claim that exceeds this
section is a lie.

### 2.1 Adversaries

| # | Adversary | Capability assumed | Addressed by |
|---|---|---|---|
| **A-hub** | The hub operator (Lockamy Studios) | Reads and retains every envelope, every payload, all metadata, for 7 days. Can substitute any value it serves, including roster entries. Can drop, delay, and reorder. | E1 (§6) for content; §5's attestation chain for key substitution. **Not addressed: metadata, drop, reorder.** |
| **A-hubx** | A compromised hub | As A-hub, plus arbitrary code | Identical treatment. The design gives the honest hub no privilege the compromised one lacks — that is the point. |
| **A-device** | A currently-paired device on the same household | Holds the content key, holds valid capabilities | **Not addressed at E1, by design.** Intra-household isolation is E2. The v1 grant model says everyone in the house sees household content; the key model agrees with it rather than contradicting it. |
| **A-revoked** | A previously-paired, now-revoked device | Retains every key it ever held, retains its 7-day catch-up reach | **Not addressed until E3.** Revoking a device stops future *grants*; it does not un-give a key. §9 is the honest statement. |
| **A-net** | A network observer | Sees TLS-wrapped traffic to the hub | Addressed by transport TLS (rustls, WSS). Frame sizes and cadence still leak — see 2.3. |
| **A-host** | A same-uid process on the gateway host | Reads gateway memory and files | **Not addressed.** Endpoint compromise is out of scope (2.3). Secure-storage rules (§3.5) raise the bar; they do not close it. |
| **A-guest** | A guest grantee holding a capability but no account | Holds exactly what was granted | E2 (per-resource keys). **A guest MUST NOT receive the household content key** — §3.4. |
| **A-bootstrap** | Whoever can read the bootstrap container's memory during the handoff window | The pre-handoff authority root | Addressed by **authority-root rotation at handoff** (§4.3). A key scraped from the bootstrap container authorizes nothing after handoff. |

### 2.2 In scope

- **Content confidentiality against the hub** — the hub must not be able to read
  household content. This is E1 and it is the reason this document exists.
- **Key-distribution integrity** — the hub must not be able to substitute a key
  and thereby read content or receive a wrap.
- **Secret transport** — an install credential must be able to cross the fabric
  without the hub reading it. This is E0.
- **Fail-closed enforcement** — an absent, malformed, expired, or unverifiable
  key or wrap denies. There is no plaintext fallback anywhere in this document.

### 2.3 Explicitly out of scope, named plainly

Stated here so nobody has to infer it from silence, and so the public copy has a
source of truth to match.

- **Metadata is permanently visible to the hub, and encryption does not change
  that.** Topic, sender device, sender account, message id, server timestamp,
  frame size, and publish cadence are all in the clear, forever, by construction
  of a hub-anchored architecture. **Publish cadence on a household smart-home
  feed is a direct occupancy signal.** This is not a temporary acceptance
  pending some later release; there is no release that fixes it. It belongs in
  public copy, not in a footnote.
- **Availability.** A hub that drops, delays, reorders, or refuses is not
  defended against by anything here. Encryption is orthogonal to delivery.
- **Traffic analysis.** Inferring household activity from ciphertext sizes and
  timing is possible and undefended.
- **Endpoint compromise.** If a device is compromised, its keys are compromised.
  Secure-storage requirements (§3.5) exist to prevent *casual* exposure — a
  backup sweep, a log line, a world-readable file — not to defeat an attacker
  with code execution on the device.
- **Forward secrecy and post-compromise security.** A content key compromised
  today decrypts everything encrypted under it, including everything already
  published, for the whole epoch. There is no ratchet. That is E4 and it is not
  in this design.
- **Intra-household confidentiality at E1.** Every paired device holds the same
  content key. Deliberate, and matched to the grant model.
- **The grant graph.** Capability tokens are signed, not sealed. The hub can see
  who was granted what (§8.2).

### 2.4 The honest sentence, and when each becomes true

The rule is: **claim custody, never capability.** "We do not keep your keys" is
verifiable by reading the code. "We cannot get your keys" requires hardware
attestation this design does not have.

| Sentence | May be published |
|---|---|
| "Your household key is generated on your own device and is never sent to us." | **When the bootstrap rework lands** (§4.3). Not before — today `api/src/device0.rs` generates a household keypair server-side and persists the seed to DynamoDB. |
| "We do not store your keys. There is no key database." | Same gate, and only once `device0_seed_hex` is **deleted**, not merely unused. |
| "Nothing you publish is readable by us." | **When E1 ships** (§6), and only for content published after the household's content key exists (§11.2). |
| "We cannot access your household." | **Only after authority-root rotation at handoff** (§4.3). Without rotation this sentence is false — the operator can read container memory while the bootstrap container lives. |
| "It is cryptographically impossible for us to read your data." | **Never, under this design.** It would require hardware attestation (SEV-SNP / TDX / Nitro Enclaves) or a bootstrap that does not run on operator infrastructure. Neither exists here. |
| Any present-tense claim of end-to-end encryption | **Not until E1 ships.** Any such sentence live today is false. |

---

## 3. Key hierarchy

The load-bearing section. Five key types. Two of them share a household scope
and are constantly confused; §3.3 exists to make that confusion impossible.

```
household authority root   Ed25519 signing     ── delegates by CHAIN-SIGNING
  │                                                (Biscuit root; §4.1)
  ├── gateway delegate token
  ├── device delegate token
  └── guest capability

household content key      XChaCha20-Poly1305  ── delegates by HPKE WRAP
  │                        symmetric, 32 bytes     (§3.3; never derived)
  └── one wrapped copy per holder device

device identity key        Ed25519 signing     ── per device, exists today
  └── cross-signs ──▶ device sealing key   X25519  ── per device, receives wraps
```

**These two households keys never touch.** The authority root cannot decrypt
anything. The content key cannot authorize anything. Neither is derived from the
other, ever.

### 3.1 Device identity key (Ed25519) — implemented

Each device holds an Ed25519 keypair generated **on the device**, never
transmitted. `gateway/identity.rs` already does this correctly: it generates its
own seed on first boot and never sends it anywhere. `fabric-kit`'s
`DeviceKeypair` is the client-side equivalent.

It is the device's name in every trust decision in this document: the pairing
attestation is signed by it, the sealing key is cross-signed by it, every
content-key grant is signed by it, and every capability names it.

**Normative:** the identity private key MUST NOT leave the device. It MUST NOT
be converted, by the Ed25519↔X25519 birational map or by any other means, into a
key-agreement key. See §3.2.

### 3.2 Device sealing key (X25519)

A **separate** X25519 keypair per device, used only as an HPKE recipient key. It
is what a content-key wrap or a sealed secret is addressed to.

**Normative — when it is minted.** A device MUST mint its sealing keypair **at
the same moment it mints its identity keypair**, before its first `THello`,
before it claims a pair session. Not at pairing. Not lazily on first
wrap-request.

The reasoning is not convenience:

- The sealing public key must be present in the pairing claim so the **approving
  device's F6 attestation signature covers it** (§5.3). A sealing key that
  arrives after the attestation has no trust path back to the human tap, and the
  only remaining source is the hub — which is exactly the substitution attack
  §5 exists to prevent.
- Lazy minting creates a state in which a device is paired but unsealable.
  Something must then decide what to do with a message for a device with no
  sealing key, and every convenient answer is a fail-open. Removing the state
  removes the question.

**Normative — cross-signature.** The sealing key is meaningless until bound to
the identity key. The binding is:

```
sealing_binding_sig = Ed25519_sign(
    identity_sk,
      "qr-sealing-binding-v1"            // 21-byte ASCII context
    ‖ 0x00                                // domain separator terminator
    ‖ len16(device_id) ‖ device_id_utf8
    ‖ x25519_pubkey                       // 32 bytes, RFC 7748 encoding
    ‖ be64(created_at_ms)
)
```

Every field that is variable-length is length-prefixed. The context string and
the `0x00` terminator are mandatory and are not decoration: **the existing F6
pairing attestation signs a bare 32-byte Ed25519 public key with no domain
separation**, so without a context string here, a signature produced for one
purpose could be presented as a signature for the other. Every signature defined
in this document is domain-separated for that reason, and the F6 material should
be domain-separated too when its attestation is revised (§5.3, **F7**).

**Normative — why not a birational conversion of the Ed25519 key.** The
Ed25519→X25519 map is real and it is tempting because it removes a key. Reusing
one keypair across a signature scheme and a key-agreement scheme means the
security argument for each depends on the other's implementation details, and
the resulting cross-protocol analysis is subtle, easy to get wrong, and
unnecessary. **One extra 32-byte key is cheaper than the argument.** This was
ruled explicitly in the capability model and is restated here as the
authoritative home.

### 3.3 The rule: never derive, always wrap

> **A content key MUST be generated by a CSPRNG, independently of every other
> key in the system. It MUST NOT be derived — by KDF, by hash, by truncation, or
> by any other function — from the household authority root, from a device
> identity key, from a parent content key, or from anything else.**
>
> **A content key reaches a holder in exactly one way: HPKE-wrapped to that
> holder's device sealing key.**

Three reasons, in the order they matter:

1. **Derivation destroys E2 before E2 is designed.** If content keys were
   HKDF-derived from a household root, anyone holding the root derives every
   child. Per-resource isolation — the entire point of E2, and the constraint
   the hosted-gateway security gate turns on — would be impossible to add later
   without changing the root, which is to say without a full re-key of every
   household.
2. **Derivation collapses the two axes that must stay separate.** Role (who may
   approve, grant, delegate) and content-key membership (who can decrypt) are
   independent by design. Deriving the content key from the authority root makes
   *granting root* silently equal *granting retroactive decryption of
   everything, permanently*.
3. **It makes the bootstrap window catastrophic instead of merely bad.** One
   memory read of the bootstrap container would yield forward
   capability-minting **and** all future content decryption for the life of the
   household.

**Mechanical tripwire, and it is deliberate:** there is **no key derivation
anywhere in E1**. `fabric-kit` therefore does **not** take a dependency on
`hkdf`, and does not need `sha2` directly. If a pull request adds `hkdf` to
`fabric-kit`, that is the signal to re-read this section before approving it.

### 3.4 The household content key

**E1 uses exactly one content key per household.** It is not a special key type;
it is a resource key whose grantee set happens to be "every paired device." That
framing is the whole reason the E1/E2 cut line is safe to take: **E1 is E2 with
N = 1.** Going to E2 changes a cardinality and a lookup, not an architecture.

| Property | Value |
|---|---|
| Algorithm | XChaCha20-Poly1305 (§6.2) |
| Length | 32 bytes |
| Generation | CSPRNG (`OsRng`), **on user-1's first real device, after the authority handoff completes** (§4.3) |
| Identity | `key_id` — 16 random bytes, minted with the key (§3.5) |
| Epoch | `u32`, `0` at birth; bumped only by E3 (§3.6) |
| Scope | one household / account, all topics under `account/<account_id>/…` |
| Holders | every device admitted through §5.6 |
| Delegation | HPKE wrap to a verified device sealing key. **No other mechanism.** |
| Export | **Never.** Not displayed, not printed, not backed up to any cloud service, not included in a diagnostic bundle, not written to a log at any level. |
| Death | when the last holder loses it, the household's E1 content is unrecoverable |

**Normative — where it is born.** The content key MUST be generated on user-1's
first real device, after the authority-root handoff (§4.3) has completed. **The
bootstrap container (device-0) MUST NOT generate, receive, hold, or wrap the
content key, at any point, under any circumstance.**

This is not a style preference. The argument that makes an operator-run
bootstrap container acceptable at all is that *the household is empty while it
lives* — there is no content to decrypt. That argument covers content that does
not exist. It does not reach forward in time. A content key minted in the
bootstrap container encrypts content at t=1…∞, and a copy taken from container
memory at t=0 decrypts all of it. **Generating the content key in the bootstrap
container puts the operator back in possession of household plaintext with no
categorical defence.**

**Normative — guests.** A principal outside the household MUST NOT be admitted
to the household content key. Guest sharing requires per-resource keys (E2); it
is not achievable at E1 by admitting a guest device, and any implementation that
does so has silently converted a guest grant into full household access.

### 3.5 `key_id` from day one, even at N = 1

Every content key carries a 16-byte `key_id`, present in every ciphertext header
from the first E1 frame, even though there will only ever be one value per
household until E2.

- **A key you cannot name is a key you cannot rotate.** Same argument as
  `grant_id` in the capability freeze.
- Catch-up (§6.6) must be able to select a decryption key by identifier, because
  a frame from the store may predate the key the device currently considers
  current.
- E2 is then a cardinality change: more `key_id` values, same header, same
  lookup.

**Normative:** `key_id` MUST be 16 CSPRNG bytes generated alongside the key. It
MUST NOT be a hash, truncation, or any other function of the key material — that
would make the header a 128-bit oracle on the key and would put a key-dependent
value in the clear on every frame.

**Storage of record.** Content keys and sealing private keys live in platform
secure storage: `flutter_secure_storage` on Courier (iOS Keychain / Android
Keystore), the OS keyring or a mode-0600 file under `$XDG_STATE_HOME` on a
gateway. Never `shared_preferences`. Never an environment variable — a secret in
`Environment=` is visible to `systemctl show`. Never a file inside a directory
that a backup or sync tool sweeps by default.

### 3.6 Key epochs — defined now, unused until E3

The `epoch` field (`u32`) exists in the ciphertext header and in the content-key
grant from the first E1 frame. At E1 it is always `0`. E3 bumps it on
rekey-on-revoke.

**Normative retention rule, and it is a constraint on the record, not on the
code:**

> **A holder MUST retain the decryption key for every epoch whose frames may
> still be reachable through catch-up: retention ≥ the catch-up horizon (7 days)
> past the epoch's retirement.**

Discard an epoch key sooner and catch-up across the boundary silently fails.
Discard it later and revocation means less than the copy says. The horizon is
the number, in both directions.

**Fail-closed requirement:** a frame naming an epoch the device does not hold
MUST surface as a decryption failure. It MUST NOT be silently dropped, MUST NOT
be rendered as an empty or partial message, and MUST NOT be returned as garbage
bytes to the application layer.

---

## 4. Root of authority vs. root of custody

Two different questions that a single phrase — "the household root key" — has
repeatedly collapsed into one. **That phrase is retired.** The two objects are
`household_authority_root` and `household_content_key` and they are never named
alike.

| | `household_authority_root` | `household_content_key` |
|---|---|---|
| Primitive | Ed25519 signing keypair | XChaCha20-Poly1305 symmetric key |
| Public half | published; every verifier needs it | none exists |
| Purpose | roots the capability chain | encrypts payloads |
| Delegates by | **chain-signing a narrowing block** | **HPKE wrap to a sealing key** |
| Held by | root-permission principals (private); everyone (public) | every member of the grantee set |
| Compromise | mints capabilities **forward, forever** | decrypts **backward**, for that epoch |
| Rotation | new root pubkey, re-issue tokens | epoch bump + rewrap (E3) |
| Derivation | chain-signing is correct and is the point | **never derive — always wrap** |

Note that the last row is an *inversion*. For the authority key, chained signing
is the correct delegation mechanism and wrapping would be nonsense. For the
content key, wrapping is correct and derivation is the defect §3.3 exists to
prevent. **A single key that does both is simultaneously wrong in both
directions.**

### 4.1 Root of authority

> **The root of authority is the household authority root. Account membership
> confers IDENTITY, NOT ACCESS.**

A gateway's device key is the **holder of a delegated, attenuable, revocable
capability** over the namespace it serves — not a sovereign root of its own.
This matters because one of the gateways may be operator-run: under
per-gateway roots, an operator-run gateway would hold unattenuated,
unrevocable authority over whatever it serves, and "relay-only" would be a
promise rather than a property. Under a single household root, an operator-run
gateway's scope *is itself a token* — expiring, attenuable, revocable by the
household.

**Consequence for pairing.** Pairing a device grants it a `device_id`, a roster
entry, and a transport session. It grants **zero** access to any namespace,
because the newly paired device holds no capability rooted in the household
authority root. Access arrives only as a separate, explicit, attenuated,
expiring token issued after a human decision. This is what bounds transitive
pairing — not a policy check inside `approve()`.

**And, critically, pairing does not confer content-key membership either.** See
§4.5.

### 4.2 Root of custody — there is none

> **There is no root custodian of the content key. Every holder keeps its own
> wrapped copy, and every copy is equal.**

Two reasons:

1. A sole custodian is a single point whose loss loses the household's data.
2. A sole custodian is, structurally, exactly the "never sole custodian"
   condition that the hosted-gateway security gate refuses to clear. Designing
   one in makes that gate permanently unclearable.

### 4.3 D1 resolved — the bootstrap container mints authority, never content

The blocking question for E1 was: does a Lockamy-operated principal hold
household key material? The answer is **no**, and it is categorical rather than
a matter of degree, for two independent reasons that must both hold.

**The bootstrap sequence (normative):**

1. A containerized gateway starts **with no key**, generates a household
   authority root, and spawns the household. This is device-0.
2. device-0 is plumbed to accept exactly the user who called it into existence.
   That user is onboarded and granted root.
3. **user-1's device generates its own household authority root, locally.**
4. **device-0 signs a one-time transition attestation** binding old-root →
   new-root, and publishes it.
5. **device-0 dies** (§4.4).
6. **After handoff completes, user-1's device generates the household content
   key** (§3.4) with a fresh CSPRNG draw, independent of everything above.

Why both rotation *and* post-handoff content-key generation are required:

- **Rotation (step 3–4) is what makes the authority claim cryptographic rather
  than custodial.** Ephemerality is no mitigation for a signing key: one
  exfiltrated from container memory during a thirty-second window is exactly as
  valid three years later, and nothing in the system would ever detect it.
  "Dies" describes the container, not the key's usefulness to whoever copied it
  first. After rotation, anything scraped from device-0 authorizes **nothing**.
- **Post-handoff content-key generation (step 6) is what makes the
  confidentiality claim hold forward in time.** See §3.4.

The construction in step 4 is deliberately the same shape as the F6 pairing
attestation (an outgoing authority signs an incoming key), one layer up. That
this primitive now appears three times — pairing attestation, authority
transition, content-key grant — is a signal it is the right one, and all three
should share a signature-material discipline (context string, `0x00`, length
prefixes).

**Transition attestation material:**

```
transition_sig = Ed25519_sign(
    old_root_sk,
      "qr-authority-transition-v1"
    ‖ 0x00
    ‖ len16(account_id) ‖ account_id_utf8
    ‖ old_root_pubkey                     // 32 bytes
    ‖ new_root_pubkey                     // 32 bytes
    ‖ be64(signed_at_ms)
)
```

A verifier presented with a capability chain MUST accept a chain rooted in
`old_root_pubkey` **only** if it can also produce this attestation, and MUST
reject any capability *issued after* `signed_at_ms` under the old root. The
old root is a dead end, not a co-equal.

**Fail-closed on a stalled bootstrap.** device-0 MUST carry a short TTL —
minutes, not hours. If handoff does not complete before expiry, the household
MUST be **destroyed, not orphaned**. A household whose bootstrap expired
mid-flight and whose record survives has zero root holders and zero key holders:
bricked at birth, arrived at by accident. Fail closed means discard, not
half-create.

### 4.4 What "device-0 dies" actually requires

Process exit is **not** the death event. Three things are, and only the first
does not follow automatically from the container being gone:

1. **Credential destruction.** The hub-issued bearer token
   (`qr:authtoken:<token>`, Redis-backed, **no TTL**) MUST be deleted, and the
   persisted copy (`device0_bearer_token` in DynamoDB) MUST be removed. Killing
   the container closes the socket and leaves that token valid forever — a
   closed door with the key under the mat.
2. **Roster removal.** device-0's entry MUST be removed from
   `qr:deviceroster:<account>`. Today this is cosmetic. **Under E1 it is a
   silent E2E break**: any client wrapping the content key to "every roster
   member" would wrap to a dead, operator-generated key. This is an ordering
   constraint, not a cleanup task — see §11.3.
3. **Seed deletion.** `device0_seed_hex` MUST be deleted from DynamoDB, not
   merely unused. A seed column that exists is replicated, snapshotted by
   point-in-time recovery, carried into every backup, reachable by any IAM
   principal with table read, and producible under subpoena. **A secret that
   does not exist cannot be breached, leaked, misconfigured, or compelled.**

**The user-confirmable artifact is the roster, not the process.** A user cannot
verify that a container exited on operator infrastructure. They *can* verify
device-0's absence from their own roster, read from their own device and
verified against their own attestation chain. Anything else is a promise taken
on faith.

`DSessionRevoked` (envelope field 105) is **not** required for device-0's death
— process exit removes the peer, which is strictly stronger than asking it to
stop. 105 remains required for the general case: revoking someone else's
already-connected device.

### 4.5 Content-key membership is conferred by an act, never by a permission bit

The strong form of the rule, and the reason it is stated as a property rather
than a policy:

> **Admission to the content-key set is conferred only by an act that requires
> the plaintext content key — namely, HPKE-wrapping that key to the admitted
> device. It is therefore never a permission bit, and can never be delegated
> wrong or forgotten.**

Consider the alternative. If "may admit to the key set" were a capability, then
under a delegation model that explicitly permits attenuated re-delegation to
sub-users, the moment that capability reaches a principal that is **not** a key
holder, an escalation path opens silently. Making admission require the key
makes the escalation physically impossible rather than merely prohibited.

Two direct consequences:

- **Approving a pairing MUST NOT admit the new device to the content-key set.**
  These are two separate acts, potentially by two different principals, at two
  different times (§5.6).
- **An operator-run resident container (device-1) MUST NOT hold the
  pairing-approval capability**, and MUST NOT be admitted to the content-key
  set. A principal that could both approve a pairing and sit in the key set is a
  complete E1 bypass available the moment a user opts into a hosted presence,
  achieved without touching hub infrastructure at all. Re-anchor the approve tap
  on a surviving root-holding **user** device.

### 4.6 Recovery, stated without softening

Recovery is **any surviving device that holds the content key**. There is no
operator-held escrow, no recovery key held by us, and no path by which Lockamy
Studios can restore a household's content.

> **A household that loses every content-key holder loses its E1-encrypted data
> permanently. There is no recovery.**

This is the standard end-to-end trade and it must be stated **at the choice
point**, when the user is deciding what to grant and what to decline — not
discovered afterwards. The useful control is not an "at least one holder"
invariant, which is satisfied by a household whose only holder's phone is at the
bottom of a lake:

- **Hard-refuse at zero.** Revoking content-key membership from another
  principal is a boundary-crossing operation, MUST be protocol-enforced, and
  MUST fail closed if it would leave zero holders.
- **Warn loudly at one.**
- **Surface the holder count in the UI at all times.**

Dropping *your own* membership is unenforceable in principle — it is deleting a
secret from your own device, and any check would live inside the principal doing
the deleting. Document it as a user risk; do not build a control that fails open
by construction.

---

## 5. Key distribution

### 5.1 The problem

**There is no wire path today that delivers one device's public key to another.**
`RHello.presence_digest` carries `DevicePresence {device_id, state, last_seen_ms,
client_info}` — no key. The roster (`qr:deviceroster:<account>`) is hub-side
Redis and is not served to clients.

Every part of this document needs that path: a capability issuer needs the
grantee's key to mint a token, and every wrap in §§5.6, 6, 7 needs the
recipient's sealing key.

**This was filed as D3 and it is now ruled.** The wire path is `TRoster` /
`RRoster` at envelope fields 50 / 51, specified in §5.4. `presence_digest` is
**not** extended and `RHello` is **not** changed; §5.4.6 records why. What
follows in §§5.2–5.3 is the trust contract that path satisfies — this
document's to specify — and §5.4 is the path itself.

### 5.2 The pairing attestation (F6) — the household-local PKI

`pairing::approve()` verifies the approving device's Ed25519 signature over
`new_device_pubkey`. Since commit `8c63496` the hub persists it at
`qr:deviceattestation:<account>:<device>` with `approver_device_id`,
`approver_pubkey` (snapshotted at signing time), `signature`,
`new_device_pubkey`, and `signed_at_ms`.

**Nothing verifies a chain of these yet.** That verification is what turns the
roster from "whatever the hub says" into a chain of device-signed attestations
terminating at the household authority root — a household-local PKI that removes
the hub from the key-distribution trust path.

**Normative:** a client MUST NOT accept a peer's public key on the hub's word.
It MUST verify an attestation chain from that peer back to a key it already
trusts, terminating at the household authority root (§4.1), accepting the
transition attestation (§4.3) where the chain crosses the handoff.

Devices paired before F6 shipped have no attestation. `None` is a permanent,
expected state for them, not an error — and it means **they cannot be verified
and therefore cannot be wrapped to.** See §11.4.

### 5.3 F7 — the attestation must cover the sealing key

**A new freeze item, and a closing door.**

F6 as landed signs **only** `new_device_pubkey` — the Ed25519 identity key. It
does not cover the X25519 sealing key, because no sealing key exists yet. That
means that as specified today, a device's sealing key would arrive with **no
path back to the human tap**, and the only entity asserting it would be the hub
— reintroducing precisely the substitution attack the attestation exists to
prevent, at the exact layer where substitution buys the attacker plaintext.

**Freeze F7:**

1. `DPairClaim` gains `bytes new_device_sealing_pubkey` and `bytes
   new_device_sealing_binding_sig` (§3.2). Additive fields, no envelope change.
2. The approver's attestation material becomes v2 and is domain-separated:

```
attestation_sig = Ed25519_sign(
    approver_identity_sk,
      "qr-pair-attest-v2"
    ‖ 0x00
    ‖ len16(new_device_id) ‖ new_device_id_utf8
    ‖ new_device_ed25519_pubkey            // 32 bytes
    ‖ new_device_x25519_pubkey             // 32 bytes
    ‖ be64(signed_at_ms)
)
```

3. The v1 (bare-pubkey) attestation remains verifiable for already-attested
   devices, discriminated by a stored version byte. It attests the identity key
   only; a v1-attested device's sealing key must be attested separately (§11.4).

**This is a one-way door and it closes on every pairing that happens
meanwhile.** Every device paired between now and F7 produces a roster entry
whose sealing key can never be retroactively covered by the approver's
signature, because the approver signed material that did not contain it and
cannot be asked to re-sign the past. → **messaging-architect** for the wire
fields, urgently.

### 5.4 Roster read (D3) — the wire path and its trust rules

**Ruled by messaging-architect, 2026-08-17.** `protocol.md` remains normative
for the wire shape; the message definitions are reproduced here because the
trust rules are unreadable without them.

A device learns a peer's public keys by asking the hub for the peer's roster
entry and then **verifying that entry against the household authority root it
already holds**. The hub is a *directory*, never an *authority*: it stores and
returns records it cannot forge.

#### 5.4.1 `TRoster` / `RRoster`

Two additive arms on the existing envelope `oneof` — fields **50** and **51**,
with **52–59 reserved** for this family. No existing message changes. The
protocol goes from 22 envelope variants to 24.

> The ruling as first drafted said "21 to 23". That count predated
> `DSessionRevoked` (envelope 105), which was already on `main` when this was
> written. The implemented count is **24** = 9 T/R pairs + `RError` + 5
> D-pushes, against the ~30 ceiling. Corrected here so this document and
> `protocol.md` agree on the number.

```protobuf
message TRoster {
  // Device ids to fetch. EMPTY MEANS THE WHOLE ROSTER for this
  // connection's account.
  //
  // There is deliberately no account_id field. The account is the
  // session's, established at THello. Accepting one here would be an
  // authorization decision taken from client-supplied data.
  repeated string device_ids = 1;
}

message RRoster {
  // One entry per device the hub could answer for. A requested device id
  // with no entry is ABSENT, not an error; partial answers are normal and
  // MUST NOT fail the request.
  repeated RosterEntry entries = 1;

  // The household authority root every attestation chain in this reply
  // terminates at, and — where the household has been through the
  // device-0 handoff (§4.3) — the transition attestation authorizing it.
  // Both travel in the same reply: a chain that cannot be terminated
  // cannot be verified, and a verifier needing a second round trip to
  // finish checking one answer eventually ships with that call skipped.
  bytes household_authority_root_pubkey = 2;
  optional TransitionAttestation authority_transition = 3;

  // True only when the WHOLE-ROSTER form was cut short. Never true for a
  // request that named device_ids explicitly. See §5.4.4.
  bool truncated = 4;

  int64 server_time_ms = 5;
}

message RosterEntry {
  string device_id = 1;

  // Ed25519 identity key (§3.1). Always present.
  bytes ed25519_pubkey = 2;

  // X25519 sealing key and its §3.2 binding. Present only once the device
  // has minted one. ALL THREE ARE PRESENT OR ALL THREE ARE ABSENT: a
  // sealing key without its binding signature, and without the timestamp
  // that is inside that signature, is unverifiable — and an unverifiable
  // field is worse than a missing one, because something will eventually
  // use it.
  bytes x25519_pubkey         = 3;
  bytes sealing_binding_sig   = 4;
  int64 sealing_created_at_ms = 5;

  // The F6/F7 pairing attestation, verbatim as the approver signed it.
  // Absent for pre-F6 devices and for the household's first device, which
  // is attested by the authority root itself rather than by a peer.
  optional PairingAttestation attestation = 6;
}

message PairingAttestation {
  // 1 or 2 (§5.3). A stored record with no version is v1 by definition.
  // An UNKNOWN version DENIES — it is never trial-verified against each
  // known material in turn.
  uint32 version            = 1;
  string approver_device_id = 2;
  // The approver's key AS IT STOOD AT SIGNING TIME — snapshotted, not
  // looked up live, so a later rotation on the approver's device cannot
  // retroactively reinterpret history.
  bytes  approver_pubkey    = 3;
  bytes  signature          = 4;
  int64  signed_at_ms       = 5;
}

message TransitionAttestation {
  string approver_device_id              = 1;  // "device-0" today
  bytes  approver_pubkey                 = 2;  // the OLD root, at signing
  bytes  household_authority_root_pubkey = 3;  // the NEW root
  bytes  signature                       = 4;
  int64  signed_at_ms                    = 5;
}
```

One error code, in the roster decade: `ROSTER_RATE_LIMITED = 50`, with
51–59 reserved.

**Envelope field 106 is reserved for a future `DRosterChanged` push and is
deliberately not defined in v1** (§5.4.6, item 6).

#### 5.4.2 The attestation does not repeat the keys it signs — and must not

`PairingAttestation` carries the signature and the signer. It does **not**
carry its own copy of `new_device_pubkey` or `new_device_x25519_pubkey`. The
verifier reconstructs the §5.3 v2 signed material from the **entry's own**
`device_id`, `ed25519_pubkey` and `x25519_pubkey`.

This is the load-bearing property of the whole design: **a hub that
substitutes a key into a `RosterEntry` necessarily breaks the signature it is
serving in the same message.** No consistent lie is available to it. If the
attestation carried its own copy of the keys, a hub could serve a matched
(attested-key, substituted-key) pair and leave the verifier to notice a
discrepancy it may not have been written to check. **Do not add those fields
back for "completeness."**

#### 5.4.3 What the hub can and cannot do in this role

**It CAN:** withhold an entry or the whole roster; serve a stale entry or roll
back to an earlier one; delay, reorder, or refuse; and observe *which device
asked about which device* — the wrap graph, already named as a known metadata
leak in §8.2.

**It CANNOT:** substitute a key (§5.4.2); fabricate a device (there is no
approver signature chaining to this household's root); transplant an
attestation from another household (its approver chains to *that* household's
root); upgrade a v1 attestation to v2 (different signed material, and the
approver cannot be asked to re-sign the past); or make a stripped sealing key
look present (all-three-or-none, and a v1 attestation is a **denial** rather
than a downgrade — the device simply never gets wrapped to, per §5.3).

**Withholding and staleness are denial, not substitution.** That is the honest
boundary of this design, and it is the boundary of every key directory without
a transparency log. Forcing a directory to *prove* freshness is the key
transparency problem, which is **E4** and is out of scope here (§2.3). It is
named rather than papered over.

#### 5.4.4 Normative rules

**Fetch before wrap.**

> A roster entry MUST NOT be taken from any cache when it is being used to
> produce a wrap or a seal. The `TRoster` → verify → wrap sequence is one act.

Caching an entry for display, for a device list, for a fingerprint screen, or
as an intermediate in a chain being verified is fine. Caching it to decide
where a content key goes is forbidden. This is stated instead of a staleness
TTL because every such number is arbitrary and every arbitrary number is
eventually raised by whoever is debugging a slow test. The cost is **one round
trip at the moment a human is already tapping Approve**, once per device,
ever.

It also closes the one security-relevant staleness case: a device removed from
the account yesterday must not receive today's content key because someone's
cache still lists it. A cooperative hub answers correctly; a hostile hub can
only withhold, which fails closed.

**Truncation is not absence.**

> `truncated = true` MUST NOT be used to conclude anything about a device that
> is not present in the reply.

A wrap decision MUST rest on either a reply to a request that named
`device_ids` explicitly, or a whole-roster reply with `truncated = false`.
"I asked for the roster, that device wasn't in it, therefore it isn't paired"
is exactly the reasoning a truncated reply invalidates, and exactly the
reasoning that drops a device out of a household's key set.

**Fetch the whole roster when you are about to wrap (SHOULD).** A
`TRoster(device_ids = ["B"])` discloses to the hub that A is about to seal
something to B. Requesting the whole roster hides the target. If the reply
comes back `truncated`, the client falls back to the explicit form and accepts
the disclosure — stated as a trade rather than hidden, though at realistic
household sizes truncation does not fire.

**Fail closed** — unchanged from the rules this section previously carried,
and now attached to a concrete shape:

- A roster entry that fails any check MUST be treated as **absent**, not as
  degraded. No wrap is produced for it, no seal is addressed to it, no content
  key reaches it.
- Failure MUST NOT fall back to trusting the hub-served value.
- Failure MUST NOT fall back to publishing in plaintext.
- An unknown attestation version, an unknown sealing-binding context string, an
  unknown `hpke_suite`, or an unrecognized key encoding MUST deny. Never "try
  anyway."
- A device with no attestation at all (pre-F6) MUST be treated as unverifiable
  and therefore not wrappable. This is a real migration cost, and §11.4 names
  it rather than hiding it behind a fallback.

#### 5.4.5 What a reader MUST verify, in order, failing closed at each step

1. `RRoster.household_authority_root_pubkey` equals the root this device
   already holds. **If it does not**, and `authority_transition` is present,
   verify the transition attestation (§4.3) under the *old* root and accept the
   new root only through it. A root that arrives with no chain to a root
   already held MUST deny; a new root is never accepted on the hub's word.
2. For each entry, `attestation` is present. Absent → the device is
   unverifiable → absent (§11.4).
3. `attestation.version` is a version this implementation supports.
4. `attestation.approver_pubkey` belongs to a device whose own chain
   terminates at the household authority root, accepting the §4.3 transition
   where the chain crosses the handoff. Unterminated → deny.
5. `attestation.signature` verifies under `approver_pubkey` over the material
   for its version, **reconstructed from this entry's own `device_id`,
   `ed25519_pubkey` and `x25519_pubkey`** (§5.4.2). Mismatch → deny.
6. If a sealing key is to be used: all three sealing fields are present, and
   `sealing_binding_sig` verifies under this entry's `ed25519_pubkey` over the
   §3.2 material including `sealing_created_at_ms`. Missing or failing → the
   entry has no usable sealing key → no wrap.
7. For a v1 attestation, step 6 is **not sufficient**: a v1 attestation does
   not cover the sealing key, so the binding proves only that the device
   asserted the key to itself, not that a human approved it. A v1-attested
   device is **not wrappable** (§5.3, §11.4).

Any failure: the entry is discarded, the peer is treated as absent, and the
failure is surfaced. It is never downgraded to a warning and never retried
against a relaxed check.

#### 5.4.6 What was rejected

1. **Extending `presence_digest`.** A category error. `DevicePresence` is
   volatile and TTL-backed; the roster is permanent
   (`qr:deviceroster:<account>` has no TTL). Merging them forces a join between
   two lifetimes on every presence event and re-ships every device's key
   material on every presence *flap*. Worse, it makes a presence bug into a
   key-distribution bug, in a record that arrives unsolicited where a client
   has no natural place to hang verification. **`DevicePresence` and
   `DPresence` are unchanged by this ruling.**
2. **A new field on `RHello`.** `RHello` fires once, at connect. A device that
   pairs *after* your `RHello` is invisible until you reconnect — and the
   minute after a device pairs is exactly when its key is needed. It also
   charges every client the whole roster on every connect, including clients
   that will never seal anything.
3. **A `roster_version` / `roster_digest` cache hint on `RHello`.** Rejected as
   premature. Its only consumer is a cache heuristic, and a hostile hub can lie
   about it freely. Adding a field to the one message every implementation must
   parse forever, to save an occasional small round trip, is the wrong trade
   today. It is an additive field later if real telemetry justifies it.
4. **A `qr.view/invoke` payload at `/roster/read`, answered by the hub.** The
   most attractive wrong answer: it costs zero envelope change and reuses the
   §5.6 machinery. The difference is *who answers*. §5.6's key request is
   device→device with the hub relaying opaque bytes; a roster read is
   device→hub. Making the hub parse a `qr.view` payload creates a second class
   of "payloads the hub does read," and that class grows. The rule, stated
   generally: **the hub answers in the envelope; devices answer in the
   payload.**
5. **A roster topic with retained/last-value messages.** Only works if
   subscribing yields current *state* rather than *history*, and the topic
   model has no retained-message semantic. Adding one is a change to every
   topic in the protocol — far more expensive than two `oneof` arms — and the
   seven-day catch-up horizon is the wrong retention for a permanent record.
6. **A `DRosterChanged` push in v1.** It buys freshness a hostile hub cannot be
   forced to provide, and against a cooperative hub fetch-before-wrap already
   covers every case that matters. **Envelope 106 is reserved for it**, so the
   addition stays additive and unhurried.
7. **An HTTP endpoint on `api.quickring.me`.** A second protocol, a second auth
   path, a second thing to version, and a self-host parity hazard by
   construction.
8. **Peer-to-peer roster serving.** The records are self-authenticating either
   way, so this is only a transport choice — and it requires a peer online at
   fetch time and says nothing about the first fetch. The hub is the transport
   that is always there.

#### 5.4.7 Self-host, cost, and what must land first

**The §10.4 check, run.** `TRoster` is answered by whatever hub the household
is connected to. A self-hosted hub answers it identically out of its own
storage, with no call to anything Lockamy operates, and verification is
entirely local against the household authority root. **The check passes**: key
distribution does not depend on our roster being reachable, and a self-hosted
household gets bit-identical security properties.

**Cost.** Two envelope variants, five new messages (`TRoster`, `RRoster`,
`RosterEntry`, `PairingAttestation`, `TransitionAttestation` — the last two
are wire mirrors of records the hub already persists), one error code, zero
changes to existing messages, no envelope-format change, and no version bump.
A fully populated entry is ~320 bytes on the wire; 100 devices is ~32 KB
against a 64 KiB payload guidance and a 1 MiB envelope ceiling, so **size is
ruled out as a consideration.** The hub SHOULD cap the whole-roster form to
stay inside the connection's negotiated `max_envelope_size`; 256 entries is
the recommended cap and is where `truncated` comes from. A client speaking to
a hub that predates this receives `RError(code=UNKNOWN_MESSAGE_TYPE)` on its
tag — a clean terminal failure needing no feature negotiation.

**This ruling originally named two record-side gaps as hard prerequisites.
Both are already resolved on `hub` `main` as of 2026-08-17** (verified
directly against the repo after this section was first drafted — the
drafting process outran the repo state it was citing):

- **The persisted attestation's version field.** F7's version-discriminated
  shape (branch `freeze/f7-attestation-covers-sealing-key`) is **merged**
  (hub PR #13, `2026-08-17T21:39:29Z`). `pairing::record_pairing_attestation`
  on `main` takes a `version: AttestationVersion` parameter — this is no
  longer a blocker.
- **The reserved sealing-key storage.** F6's bare-key reservation
  (`pairing::set_device_sealing_key`) has been superseded on `main` by
  `pairing::record_device_sealing_key`, which persists the X25519 public
  key, the `binding_sig`, **and** the `created_at_ms` together — exactly
  the all-or-none shape §3.2's binding signature requires — and is only
  ever called after v2 attestation verification succeeds. No further
  record-shape work is needed for this specifically; a
  data-architect pass is still worth doing to confirm `admin.rs`'s
  `DeviceSummary`/`list_account_devices` should converge with the roster
  read rather than growing a second device-list format, but that is a
  design-quality question, not a blocker.

**The roster-read wire path is implementable now.**

### 5.5 Fingerprint verification — the human check

The fingerprint shown on the existing device during APPROVE is, in
`protocol.md`'s own words, "the user's last line of defence against a
hub-mediated attacker." Under F7 it MUST be computed over **both** keys, so that
the thing the human compares is the thing the signature covers:

```
fingerprint = SHA-256(
      "qr-device-fingerprint-v1" ‖ 0x00
    ‖ ed25519_pubkey ‖ x25519_pubkey
)  →  first 20 bytes, RFC 4648 base32 (uppercase, unpadded),
      rendered as 8 groups of 4 characters
```

20 bytes is exactly 32 base32 characters, hence 8 groups of 4. `protocol.md`
carries the same rendering and a conformance vector for it; where the two ever
disagree about the string a human is asked to compare, `protocol.md` wins.

**A fingerprint nobody can read is not a control, it is a ceremony.** The
rendering must meet WCAG AA contrast at minimum; a fingerprint currently
rendering at 2.40:1 fails that bar and fails this control. Legibility of this
string is a security requirement, not a design preference. → **ux-engineer**,
with the contrast defect in scope.

### 5.6 Admitting a device to the content-key set — the exchange

This is the concrete protocol answer to "a new device is paired; how does it get
the content key?"

**It is NOT delivered in `DPairResult`.** Three reasons, and the first is
structural:

1. `DPairResult` is minted and pushed **by the hub**. Putting the wrap there
   forces the wrap to exist at approve time, which forces the approver to be a
   key holder — the exact conflation §4.5 forbids.
2. It would require a wire change to `PairCredentials`.
3. It makes admission one-shot. If the approver is not a key holder there is no
   path at all, and if the wrap fails the pairing has already succeeded and the
   device is stranded with no retry.

Instead, admission is **its own exchange, over the existing `qr.view` family,
between two devices, with the hub relaying opaque bytes.**

```
step 1  the newly-paired device publishes a request

        qr.view/invoke  path = /keys/request
        args = { request_id, key_id?, requester_device_id,
                 requester_sealing_pubkey, sealing_binding_sig }
        topic: account/<id>/broadcast

step 2  ANY device that (a) holds the plaintext content key and
        (b) verifies the requester's roster entry per §5.4
        replies, after human confirmation (see below)

        qr.view/reply   in_reply_to = <message_id of step 1>
        payload = ContentKeyGrant   (§5.7)
        topic: account/<id>/device/<requester_device_id>

step 3  the requester verifies (§5.8) and unwraps. Fail closed at
        every step; on failure it remains KEY_PENDING and does NOT
        publish or render content.
```

**What this requires beyond the existing op-set: nothing on the envelope.** No
new envelope variant, no new payload family, no new error code. It uses
`qr.view/invoke` and `qr.view/reply` — both already ruled — with two new *paths*
(`/keys/request`, and the reply that answers it). Idempotency composes with the
existing persisted `request_id` set rather than introducing a second mechanism
or a second retention policy: a replayed key request resolves to the original
grant instead of producing a second wrap.

**Normative — the reply MUST NOT be automatic.** This is the load-bearing rule
of the whole exchange. If any online key holder auto-replies to any roster
device's request, then pairing once again confers content-key membership, just
via automation, and §4.5 is defeated in practice while remaining true on paper.

A `ContentKeyGrant` MAY be emitted only:

- **(a)** automatically, within a bounded window (RECOMMENDED: 5 minutes)
  immediately following an approval that **this same device** performed — the
  approver's own tap is the confirmation, and no second tap is needed for the
  normal one-device household; or
- **(b)** in response to an explicit human confirmation on the grantor device
  that names the requesting device and shows its fingerprint (§5.5).

Case (a) preserves one-tap UX for the case that is 95% of households. Case (b)
is the honest cost of a delegated approver, and it is a second tap that only
appears when approve and custody are genuinely held by different principals.

**No holder online.** The request sits on a topic with a 7-day catch-up horizon
and is answered whenever a holder next comes online. A requester or grantor that
was offline when the other side published recovers the missed frame via
**`TCatchUp`** on the key-request/reply topic — the single catch-up path (a
`TSubscribe` is forward-only from the moment it is issued; see `protocol.md`,
"Catch-up," QR-134). Until then the requester is in state `KEY_PENDING`:

- it MUST NOT publish content (it cannot — it has no key), and MUST NOT fall
  back to plaintext;
- it MUST NOT render E1 frames as empty; the UI must say the device is waiting
  for approval from another device, and name what to do about it;
- if no grant arrives within the catch-up horizon, the request expires and must
  be re-issued.

**There is no plaintext fallback anywhere in this exchange.** A device without
the content key is a device that does not publish content. That is the whole
rule.

### 5.7 `ContentKeyGrant` — byte layout

```
offset   size  field
   0        4  magic                    "QRK1"  (0x51 0x52 0x4B 0x31)
   4        1  version                  0x01
   5        1  hpke_suite               0x01  (see below)
   6       16  key_id
  22        4  epoch                    big-endian u32
  26       32  enc                      HPKE encapsulated key (X25519 ephemeral pubkey)
  58        2  ct_len                   big-endian u16  (= 48 for a 32-byte key)
  60        L  ct                       HPKE ciphertext of the 32-byte content key
                                        (32 plaintext + 16 Poly1305 tag)
  60+L     32  grantor_ed25519_pubkey
  92+L     64  grantor_signature
```

**Total: `156 + L` bytes = 204 bytes** at `L = 48` (a 32-byte content key plus
a 16-byte Poly1305 tag).

`hpke_suite = 0x01` denotes **RFC 9180**:

| Component | Value | RFC 9180 id |
|---|---|---|
| KEM | DHKEM(X25519, HKDF-SHA256) | `0x0020` |
| KDF | HKDF-SHA256 | `0x0001` |
| AEAD | ChaCha20-Poly1305 | `0x0003` |
| Mode | `mode_base` (0x00) | — |

HPKE parameters, both mandatory:

```
info = "qr-contentkey-v1" ‖ 0x00
     ‖ key_id (16) ‖ be32(epoch)
     ‖ len16(recipient_device_id) ‖ recipient_device_id_utf8

aad  = len16(account_id) ‖ account_id_utf8
```

`info` binds the wrap to a specific key, epoch, and recipient — a captured wrap
cannot be replayed as a grant of a different key or to a different device.
`aad` binds it to the household, so a grant cannot be moved between households.

`grantor_signature`:

```
grantor_signature = Ed25519_sign(
    grantor_identity_sk,
      "qr-contentkey-grant-v1"
    ‖ 0x00
    ‖ grant_bytes[0 .. 92+L]              // the whole record up to the signature
    ‖ len16(recipient_device_id) ‖ recipient_device_id_utf8
    ‖ recipient_x25519_pubkey             // 32 bytes
)
```

**Why an explicit signature rather than HPKE `mode_auth`.** `mode_auth` would
prove the sender holds the sender's X25519 private key by folding it into the
KEM. Rejected, for three reasons:

1. It authenticates the **sealing** key, while every other trust decision in
   this system — the roster, the attestation chain, the Biscuit root — speaks in
   terms of the **identity** key. An explicit Ed25519 signature by the identity
   key composes with all of them; `mode_auth` composes with none of them.
2. `mode_auth` produces a **non-transferable, deniable** authentication that
   only the recipient can check and nobody can verify afterwards. Admission to
   the key set is exactly the kind of act that should leave an auditable
   artifact.
3. It keeps HPKE usage to `mode_base`, the mode every implementation supports,
   which matters for a protocol we intend to publish.

Cost of the decision: 64 bytes plus 32 for the grantor's public key. Against a
64 KiB frame cap that is not a consideration, and it is ruled out as one here so
nobody re-litigates it.

### 5.8 What the recipient MUST verify, in order, failing closed at each step

1. `magic == "QRK1"` and `version == 0x01`. **An unrecognized version or suite
   denies.** Never attempt a "best effort" parse.
2. `hpke_suite` is a suite this implementation supports. Unknown suite → deny.
3. `grantor_ed25519_pubkey` resolves to a roster device whose pairing
   attestation chain verifies to the household authority root (§5.2, §5.4).
   Unverifiable grantor → deny.
4. `grantor_signature` verifies over exactly the material in §5.7, **including
   the recipient's own device id and sealing public key.** This is what stops a
   grant addressed to device A from being re-presented to device B.
5. HPKE `open` succeeds with the `info` and `aad` above, computed locally — not
   taken from the message.
6. The recovered plaintext is exactly 32 bytes.
7. If the device already holds a key for this `(key_id, epoch)`, the recovered
   key MUST equal it. A mismatch is a split-brain condition and MUST deny and
   alert, not overwrite.

Any failure: the grant is discarded, the device remains `KEY_PENDING`, and the
failure is surfaced. It is never downgraded to a warning and never retried
against a relaxed check.

### 5.9 Crate and library selection

This is what the crypto-primitives work adds to `fabric-kit`. It is deliberately
short.

```toml
# NEW
hpke              = { version = "0.12", default-features = false, features = ["x25519", "alloc"] }
chacha20poly1305  = { version = "0.10", default-features = false, features = ["alloc", "getrandom"] }
zeroize           = { version = "1", features = ["zeroize_derive"] }
subtle            = "2"

# ALREADY PRESENT
ed25519-dalek     = { version = "2", features = ["rand_core"] }
rand              = "0.8"
```

**Four new crates. That is the whole surface.**

- **`hpke` (rust-hpke), not `hpke-rs`.** Pure Rust on RustCrypto traits, with no
  backend-crate selection and no system dependency. This matches the repo's
  demonstrated and deliberate stance everywhere else — `protox` instead of a
  system `protoc`, `rustls` instead of system OpenSSL — which exists because
  Courier cross-compiles to iOS, Android, and macOS and a vendored-C dependency
  is a recurring build failure there.
- **Use `hpke`'s `Kem` API to mint and serialize the sealing keypair. Do NOT add
  `x25519-dalek` directly.** Two independent paths to the same key type is how a
  codebase ends up with two encodings of the same public key and a
  cross-implementation bug that only appears between two versions of our own
  client.
- **`chacha20poly1305` provides `XChaCha20Poly1305`** for the content layer
  (§6.2). Note the two layers use different AEADs on purpose (§6.2 explains
  why); this is correct, not an oversight.
- **`zeroize` on every key type.** `HouseholdContentKey`, the sealing private
  key, and every decrypted-key buffer derive `ZeroizeOnDrop`.
- **`subtle` for constant-time comparison** of `key_id` and any tag-adjacent
  value.
- **Notably absent: `hkdf`.** There is no key derivation in E1, by design
  (§3.3). Its appearance in a dependency diff is a review trigger.

**Version alignment, stated because it is the usual source of friction in this
exact stack:** all of the above sit on `rand_core` 0.6 — `ed25519-dalek` 2 via
`signature` 2, `hpke` 0.12, and `chacha20poly1305` 0.10 via `aead` 0.5. Do not
mix a RustCrypto generation into this set without checking `rand_core`; a
mismatch surfaces as an inscrutable trait-bound error rather than a version
conflict.

**Everything lives in `fabric-kit`.** Courier reaches it through the existing
`flutter_rust_bridge` → `rust_lib_courier` → `fabric-kit` path. **There is no
Dart crypto to write and no second implementation to keep in sync** — the same
argument that decided the capability token format.

---

## 6. Payload encryption (E1)

### 6.1 Ciphertext layout

`TPublish.payload` is already declared opaque bytes that the hub does not
interpret. **E1 therefore requires zero envelope change.** The ciphertext is
self-describing:

```
offset  size   field
   0       4   magic          "QRE1"  (0x51 0x52 0x45 0x31)
   4       1   version        0x01
   5       1   suite_id       0x01 = XChaCha20-Poly1305
   6       2   flags          big-endian u16
                                bit 0 — inner payload_type present
                                bits 1-15 — MUST be zero; a set bit denies
   8      16   key_id
  24       4   epoch          big-endian u32  (always 0 at E1)
  28      24   nonce          XChaCha20 192-bit nonce, fresh CSPRNG per frame
  52       N   ciphertext     AEAD output ‖ 16-byte Poly1305 tag
```

**Header 52 bytes + 16-byte tag = 68 bytes of overhead per frame.** (A prior
estimate said ~60; 68 is the real number and is recorded here rather than the
estimate.) Against the 64 KiB `TPublish` cap that is **0.1%**. Overhead is ruled
out as a design consideration, on the record, so nobody re-opens it.

### 6.2 XChaCha20-Poly1305, and why not AES-256-GCM

**Ruling: XChaCha20-Poly1305 for the content layer.**

The deciding argument is the **nonce**, and it is specific to this system rather
than a general preference:

**Multiple devices publish to the same topic under the same content key, with no
coordination point between them.** There is no sequencer, no shared counter, and
no place to put one — devices are frequently offline and publish independently.
That rules out counter-based nonces. Random nonces are the only workable scheme,
and a 96-bit random nonce (AES-256-GCM, or IETF ChaCha20-Poly1305) has a
birthday collision probability that becomes non-negligible at roughly 2³² frames
under a single key — reachable by a chatty household inside a year.
XChaCha20-Poly1305's **192-bit random nonce removes the problem by
construction**: collision probability stays under 2⁻⁹⁶ past 2⁴⁸ frames, which at
the protocol's 10 publishes/sec/connection cap across twenty devices is on the
order of a million years.

That matters more than usual because **AES-GCM's nonce-reuse failure is
catastrophic, not graceful.** Repeating a nonce under GCM leaks the
authentication subkey, which costs forgery for the whole key, not just
confidentiality for the two colliding messages. The multi-writer footgun and the
severity of the failure compound.

Two supporting reasons:

- **No AES hardware assumption.** The target matrix includes older Android
  ARMv7 handsets and ARM SBC-class gateway hardware. Software AES-GCM there is
  both slow and timing-fragile; ChaCha20 is fast and constant-time in pure
  software by construction.
- **One pure-Rust implementation, everywhere.** `chacha20poly1305` builds
  identically for every Courier target with no system dependency and no
  feature-detection path.

**The wrap layer (§5.7, §7) uses IETF ChaCha20-Poly1305, not XChaCha, and that
is correct.** HPKE's AEAD registry does not include XChaCha, and it does not
need to: every HPKE seal in this document is single-shot with a **fresh
ephemeral key per operation**, so the context's nonce sequence never advances
past zero and nonce reuse is impossible by construction. The multi-writer
argument that decides the content layer simply does not apply to the wrap layer.
Different constraint, different answer, deliberately.

### 6.3 Associated data — what is bound

```
AAD = header_bytes[0 .. 52]
    ‖ len16(topic)            ‖ topic_utf8
    ‖ len16(sender_device_id) ‖ sender_device_id_utf8
```

Every variable-length component is length-prefixed, so `topic = "a/b"`,
`sender = "c"` and `topic = "a/b/c"`, `sender = ""` cannot produce the same AAD.

Two properties fall out, and the second is worth having deliberately:

- **A frame captured from topic A cannot be replayed onto topic B.** The AEAD
  authentication fails. This is one of the things that must be proven under
  test, not asserted.
- **The hub cannot lie about who sent a frame without breaking decryption.**
  `sender_device_id` is stamped by the hub on `DDeliver`; the sender computed the
  AAD using its own device id. If the hub substitutes a different sender, the tag
  check fails at every recipient. The hub is removed from the sender-attribution
  trust path as a side effect of binding the AAD.

**Constraint on the hub, and it is load-bearing.** `RCatchUp.deliveries` is
`repeated DDeliver` and therefore already carries the original `topic` and
`sender_device_id` — verified, so catch-up works under this AAD unchanged.
**Normative: the hub MUST replay the original `topic` and `sender_device_id` on
catch-up and MUST NOT re-stamp, normalize, or canonicalize either.** Any
normalization silently renders every stored frame undecryptable, and the failure
would appear as "catch-up is broken" long after the change that caused it.

### 6.4 Zero envelope change

Restated because it is the reason E1 is a three-to-four-week piece of work
rather than a protocol revision: **`TPublish.payload` and `DDeliver.payload` are
already opaque bytes.** No field is added, no field number is consumed, no
version is bumped, and no installed client is invalidated. A client that does not
understand `QRE1` sees bytes it cannot parse — which is exactly what it must do.

### 6.5 `payload_type` — wrapped (D4, ruled)

`payload_type` is a routing hint on `TPublish` and is **never interpreted by the
hub** (`quickring.proto`: `payload_type` = "app-layer hint, never interpreted by
the hub"; routing is entirely on `topic`). Two options were open:

- **(a) Leave it plaintext.** Simpler; preserves client-side routing without
  decrypting.
- **(b) Wrap it inside the ciphertext** under a single outer type
  `qr.seal/box`, and carry the real type as the first field of the inner
  plaintext.

**Ruled (D4, messaging-architect, 2026-08-18): (b), wrapped.** Three reasons,
the middle one decisive:

1. **The hub has no legitimate use for it.** Routing and catch-up are keyed on
   `topic`; the hub never reads, filters, or dispatches on `payload_type` — the
   proto and `protocol.md` both say so normatively. So plaintext buys the hub
   nothing and costs the household a free traffic-class signal (location update
   vs. lock actuation vs. feed post) it can log and correlate. §2.3 already
   concedes a great deal of metadata; there is no reason to concede a field the
   relay was never designed to consume.
2. **The AAD binding already decided this.** §6.3 binds `topic` and
   `sender_device_id` — and *only* those — because they are the two fields the
   hub necessarily stamps or routes on and must therefore be able to read.
   `payload_type` is deliberately **not** in the AAD. A field that is neither
   sealed nor authenticated is a plaintext, *hub-mutable* field: leaving it on
   the envelope would let the relay both read the traffic class and rewrite it
   with no detection anywhere. Wrapping puts it inside the QRE1 ciphertext, where
   it is both hidden and covered by the Poly1305 tag. Given §6.3, wrapped is the
   only placement that is not strictly worse than absent.
3. **It is consistent with the hub-as-untrusted-directory posture** the D3
   ruling established (`d3-roster-read-wire-path.md`): the hub sees exactly what
   it must to relay, and nothing it was not built to act on.

**Wire mechanics (normative).**

- On an E2E (sealed) topic, `TPublish.payload_type` MUST be the single constant
  **`qr.seal/box`** and MUST NOT carry any other value. It conforms to the
  `qr.<family>/<verb>` grammar, classifies nothing (every sealed frame carries
  the identical string), and positively signals "decrypt before routing" so an
  SDK need not sniff `payload` bytes. `qr.seal/box` is a conventional value of
  the existing free-form string field — **no schema change, no new envelope
  variant, no version bump** (E1's zero-envelope-change property holds).
- The real application type travels inside the QRE1 frame when `flags` bit 0 is
  set, as the first field of the inner plaintext:

```
   0       1   payload_type_len (u8)
   1       T   payload_type_utf8
 1+T       M   inner payload bytes
```

- `DDeliver.payload_type` (the hub's echo, field 8) therefore carries only the
  `qr.seal/box` sentinel — or is omitted — on sealed topics. The hub MUST NOT
  synthesise or infer a real type it cannot see. `RCatchUp` inherits this
  unchanged: it returns ciphertext, the client decrypts and recovers the inner
  type exactly as for a live delivery (§6.6), so wrapping does not disturb
  catch-up.

Compatible with every other section of this document.

### 6.6 Catch-up

`RCatchUp` returns ciphertext. The client selects a decryption key by the
header's `(key_id, epoch)` and decrypts exactly as it would a live delivery —
the SDK already unifies the two paths, and E1 does not disturb that.

This is the reason for §3.6's retention rule. A frame retrieved from a 7-day
store may name an epoch older than the device's current one; if that key has
been discarded, catch-up fails. **Fail closed:** a frame whose `(key_id, epoch)`
is unavailable surfaces as an explicit decryption failure. It is never silently
skipped and never returned as garbage.

### 6.7 What E1 does and does not buy

| Buys | Does not buy |
|---|---|
| The hub stops reading household content | Any metadata protection (2.3) |
| The 7-day catch-up store becomes ciphertext | Intra-household isolation — every paired device holds the key |
| Location and media *metadata* stop being a vendor-plaintext problem | Content-bearing paths (photo bytes, camera footage) — those need E2 |
| A relay-only hosted tier becomes honestly sellable | Read revocation — the bytes already left (§9) |
| Protocol publication becomes credible | Forward secrecy — a compromised key decrypts the whole epoch, backwards |

---

## 7. The sealed-payload primitive (E0)

### 7.1 Purpose

Seal a blob to **exactly one device**, such that no other device — including
another device on the same household — and not the hub can open it. Two
consumers:

- **Remote install with secrets.** A user on a phone supplies an API token for a
  service being installed on a gateway. Today this is blocked entirely, and the
  workaround is friction: "finish setup on `<host>`."
- **Capability delivery**, where the token itself should not be readable by the
  relay.

E0 is the same HPKE machinery as §5.7 with a different `info` string and a
different payload. It is deliberately built first, because E1 consumes it.

### 7.2 `SealedBlob` — byte layout

```
offset   size  field
   0        4  magic                 "QRS1"
   4        1  version               0x01
   5        1  hpke_suite            0x01 (same suite as §5.7)
   6       16  seal_id               CSPRNG, single-use handle
  22       32  enc                   HPKE encapsulated key
  54        2  ct_len                big-endian u16
  56        L  ct                    HPKE ciphertext of the inner plaintext
  56+L     32  sender_ed25519_pubkey
  88+L     64  sender_signature
```

**Total: `152 + L` bytes**, where `L` is the secret plus the inner header plus
the 16-byte tag.

Inner plaintext (encrypted — this is the point):

```
   0        8  not_after_ms          big-endian i64, unix ms UTC
   8        2  purpose_len           big-endian u16
  10        P  purpose_utf8          e.g. "install-secret", "capability"
  10+P      S  secret bytes
```

HPKE parameters:

```
info = "qr-seal-v1" ‖ 0x00 ‖ seal_id (16)
     ‖ len16(recipient_device_id) ‖ recipient_device_id_utf8
aad  = len16(account_id) ‖ account_id_utf8
```

`sender_signature` = `Ed25519_sign(sender_identity_sk, "qr-seal-v1-sig" ‖ 0x00 ‖
blob[0 .. 88+L] ‖ len16(recipient_device_id) ‖ recipient_device_id_utf8 ‖
recipient_x25519_pubkey)`.

### 7.3 Validity is inside the ciphertext, and single-use is enforced locally

**Normative:** `not_after_ms` lives **inside** the encrypted plaintext, not in
the header. If the expiry were a header field, the hub — which holds the blob for
seven days whether we like it or not — could present it with any expiry it liked.
Inside the AEAD it is authenticated and unmodifiable.

**Normative:** the recipient MUST reject a blob whose `not_after_ms` has passed,
and MUST reject a `seal_id` it has already consumed. The consumed-`seal_id` set
composes with the existing persisted `request_id` set — same store, same
retention rule (≥ the 7-day catch-up horizon, because **catch-up is exactly the
replay path**). No second nonce store, no second retention policy.

RECOMMENDED validity for an install secret: **5 minutes**. The user is present
and watching; there is no reason for a longer window and the blob sits in a
seven-day store regardless.

### 7.4 What this retires

The ephemeral-handshake construction previously sketched for secret transport is
**retired, not reviewed.** Its weakness was structural: it had no trusted
key-distribution path, so a hub that could substitute a key could sit in the
middle of it. §5's attested roster is that path. Once E0 exists, the handshake
has nothing left to do, and reviewing a construction we intend to delete is
wasted work.

---

## 8. Capability tokens and encryption — the seam

### 8.1 Capabilities are signed, not encrypted, and that is correct

A capability token crosses the fabric in the clear. The hub reads it. **This is
safe, and it is safe for a specific reason: capabilities are holder-of-key.** A
token names a grantee public key, and every request carries a fresh
proof-of-possession signature by the matching private key over the path, the
argument hash, and the `request_id`. Copying the token gains an attacker
nothing. Hub visibility and the seven-day catch-up window therefore stop being
fatal for tokens, in a way they are not for content.

Encrypting capability tokens would buy grant-graph confidentiality (§8.2) at the
cost of requiring the issuer to know the recipient's sealing key before it can
issue — and would not improve unforgeability at all, since the signature already
provides it. Not worth it at E1.

### 8.2 What the hub therefore learns

**The grant graph.** Which key was granted which operations on which paths,
under which resource root, with what expiry. Named here as a known, accepted
metadata leak rather than left to be discovered.

E2 reduces its usefulness — when content is per-resource encrypted, knowing that
a grant exists tells you less than when one key opens everything — but **nothing
in this design makes the grant graph invisible.** Do not write copy that implies
otherwise.

### 8.3 Grant revocation and key revocation are different operations

They have different mechanisms, different latencies, and different guarantees.
**Never conflate them in code, in UI copy, or in support answers.**

| | Grant revocation | Key revocation |
|---|---|---|
| What it does | stops future *authorized operations* | stops future *decryption* |
| Mechanism | revocation-id list at the gateway, plus token expiry | epoch bump + rewrap to remaining holders (E3) |
| Latency | immediate — a local decision at the enforcement point | E3; not available at E1 |
| What it does not do | does not un-send bytes already published | does not un-decrypt frames the revoked holder already read |

A capability revoke is not a content revoke. A UI element that revokes a grant
and implies the revoked party can no longer read what they already have is
false, and this is the single most likely place for the product to make a claim
the cryptography does not support.

---

## 9. Revocation and rekey (E3 — specified, deferred to 2028)

Specified here so E1's format reserves what E3 needs. **Not implemented, not
scheduled before 2028.**

### 9.1 The mechanism

1. A remaining holder generates a **new** content key (CSPRNG, independent —
   §3.3 applies unchanged) with a new `key_id` and `epoch = previous + 1`.
2. It wraps the new key to every remaining holder via §5.7.
3. New publishes use the new `(key_id, epoch)`. Old frames stay under the old
   key.
4. Every holder retains the old epoch key for the retention window (§3.6).

**Offline holders are the hard part**, and it is why this is E3 rather than a
follow-on to E1: a device offline across a rekey returns to find frames it
cannot decrypt and must request the new epoch through §5.6's exchange — with all
of §5.6's human-confirmation requirements intact, because relaxing them for
convenience during a rekey would reopen §4.5.

### 9.2 The unavoidable catch-up tail

**Ruling: an honest seven-day tail, not purge-on-revoke.**

A revoked holder keeps the old key and keeps its reach into frames already in
the seven-day store. The alternative — purging the store on revoke — is a
hub-side deletion the household **cannot verify**, and it would be a stronger
promise than we can keep. An unverifiable deletion presented as a security
guarantee is worse than an honest limitation, because the user makes decisions
on it.

State the tail in UI copy, in those words.

### 9.3 Honest revocation SLOs, per class

| Class | SLO |
|---|---|
| Invoke authorization | **Immediate** — a local decision at the gateway, no network round trip |
| Read, future frames | **Immediate** — publishes under the new epoch are unreadable to the revoked holder |
| Read, already-published frames | **Up to 7 days** — the catch-up horizon, and there is no shorter honest number |
| Content already decrypted and stored by the revoked device | **Never.** It is theirs. No design in this document changes that. |
| Session | one heartbeat (~30 s) at the token-recheck rung; immediate with `DSessionRevoked` (field 105) |

**Do not tune the catch-up horizon as a security control.** Shortening seven-day
retention trades a real availability property — a phone that was off for a week
recovers its state — for a marginal confidentiality gain, and it does not change
the fact that the hub held the frame. It looks cheap and buys close to nothing.

---

## 10. Self-host decoupling — a verified check, not an aspiration

The lock: **a self-hosted deployment must obtain identical security properties
with no dependency on any Lockamy-operated service.** Checked against this
design, item by item, rather than asserted.

**10.1 No key material reaches a Lockamy service.** The authority root is
generated on user-1's device (§4.3). The content key is generated on user-1's
device after handoff (§3.4). Sealing keys are generated on each device (§3.2).
Every key that crosses the fabric does so HPKE-wrapped to a verified recipient
(§5.7, §7.2). The hub relays opaque bytes. **PASSES**, conditional on the
bootstrap rework landing — it does **not** pass against today's
`api/src/device0.rs`, which generates a household keypair server-side and
persists the seed.

**10.2 No licensing, feature-unlock, telemetry, or auth call.** Nothing in this
document contacts an operator service to decide whether a control applies.
Capability verification is offline against a public key the verifier already
holds. Content decryption is local. Attestation-chain verification is local.
**PASSES.**

**10.3 A self-hosted hub gets identical properties.** The hub's role under this
design is: relay opaque bytes, authorize topics against a realm-scoped
credential it *verifies but does not issue*, and hold delivery state it cannot
interpret. All three are implementable by any hub. **PASSES** — and it
strengthens self-host in passing: a hub that verifies rather than issues has **no
credential database to operate, back up, or lose.**

**10.4 The check that would fail, named so D3 does not introduce it.**

> **Any design in which a device can only learn a peer's public key by reaching
> a Lockamy-operated roster service, or in which the correctness of that key
> depends on that service being honest, breaks this lock.**

D3's roster-read path MUST be: (a) servable by any hub, self-hosted included;
and (b) **verifiable offline** against the attestation chain, so that the
serving hub is a convenience and not a trust root. §5.4's rule — a hub-served
entry that fails verification is treated as absent — is what enforces (b). If
the roster read is ever specified in a way that requires trusting the server,
this lock is broken and this section stops passing.

**Run against the ruled design (§5.4): PASSES.** `TRoster` is answered by
whatever hub the household is connected to, out of that hub's own storage, with
no call to anything Lockamy operates; and every field a verifier acts on is
covered by a signature chaining to a household authority root the client
already holds (§5.4.5). A self-hosted household gets bit-identical security
properties. The residual — that any hub, ours or theirs, can *withhold* or
*roll back* an entry — is denial, not substitution, and is named in §5.4.3
rather than assumed away.

---

## 11. Rollout and migration

### 11.1 There is no retroactive content encryption, and there must not be one

**E1 applies going forward only.** Frames published before a household's content
key exists are plaintext, were read by the hub, and sit in the catch-up store.
Encrypting them after the fact:

- cannot un-read them — the disclosure already happened;
- would require someone to re-publish them, changing message ids and ordering
  and corrupting every client's view of history;
- and is unnecessary, because the store expires them anyway.

### 11.2 The gap, stated so it can be checked

> **Every frame published before the household's content key exists is
> plaintext, was readable by the hub at publish time, and remains in the
> catch-up store for up to 7 days.**
>
> **The household's pre-E1 plaintext exposure therefore has a hard end date: 7
> days after its last pre-E1 publish. It is closed by expiry, not by us.**

That is a bounded, checkable, honest claim, and it is the right one to make. It
is also why any present-tense E2E claim published before E1 ships is false — the
gap is not merely "not yet encrypted," it is "we read it."

### 11.3 What existing households actually need — key migration, not data migration

The real migration is the keys, and it is not small. No household that exists
today satisfies E1's preconditions: none has a content key, no device has a
sealing key, and no household has run the authority handoff.

**An existing household cannot be migrated to E1 without running a retro-handoff.**
The sequence, in this order, and the order is normative:

1. **Bootstrap rework lands.** `device0_seed_hex` is deleted from DynamoDB;
   `POST /admin/device0` and the `bootstrap-device0` CLI are removed. Until this
   ships, an operator-held household key exists and no E1 claim is defensible
   regardless of what else is built.
2. **Retro-handoff.** A designated surviving user device generates a fresh
   household authority root; device-0 (or the API acting as it) signs the
   transition attestation (§4.3); the old root is retired.
3. **device-0 dies properly** — credential destruction, roster removal, seed
   deletion (§4.4). **Roster removal MUST complete before step 4.** A client
   that wraps to "every roster member" while device-0's entry survives wraps the
   household content key to a dead, operator-generated key. This is an ordering
   constraint, not a cleanup task, and getting it backwards silently voids the
   entire exercise.
4. **Content-key birth.** The same device generates the household content key
   (§3.4).
5. **Admission of every other device** through §5.6's exchange, one human
   confirmation each.

**Households that cannot complete a retro-handoff** — no surviving user device,
or device-0 already unreachable — **do not get E1.** Their honest options are
re-provision or remain plaintext.

**Do not build an operator-assisted retro-handoff.** A support tool that lets an
operator mint or rotate a household's authority root is R8's defect
reintroduced with a friendlier name: it is a standing operator capability to
take over any household, and its existence makes "we cannot access your
household" false again regardless of how rarely it is used. If a household
cannot recover itself, it cannot be recovered.

### 11.4 Devices paired before F6/F7

- **Pre-F6 devices have no pairing attestation.** They are unverifiable, and
  under §5.4 they are therefore not wrappable. They cannot receive the content
  key until they are re-attested.
- **Pre-F7 devices have a v1 attestation covering only the identity key.** Their
  sealing key — which they do not have yet, since nothing mints one — will
  arrive uncovered.

For both, the remedy is the same and there is no way around it: **the device
mints a sealing key, produces its binding signature (§3.2), and a current
content-key holder re-attests it with a v2 attestation after a human fingerprint
comparison.** That is one human tap per existing device, once. It is the cost of
F7 landing after pairing shipped, and it is why F7 should land now rather than
after another month of pairings.

### 11.5 Migrate or re-provision the alpha population — **DJ call**

The population today is a studio alpha: a small number of households whose data
is test data.

**My default is re-provision, not migrate.** The retro-handoff ceremony in §11.3
is five ordered steps with a silent-failure mode in step 3, and building and
testing it correctly for a population whose data does not matter is effort spent
in the wrong place. Re-provisioning gives every alpha household a clean
bootstrap under the new model with no ceremony and no ordering hazard.

**This is DJ's call, not mine** — it is a product and user-experience trade
(does an alpha household lose its history?), not a cryptographic correctness
question. Either answer is safe. If the answer is "migrate," §11.3 is the
specification and step 3's ordering constraint is the thing to test first.

**Design the retro-handoff anyway, later, for the case where it matters** — a
real household with real data at the point E3 or a key rotation forces the same
ceremony. Do not build it now.

---

## 12. What is NOT implemented

Deliberately duplicating §1.2's status table in prose, because a reader who
skips the table must still not come away believing more is live than is.

**Corrected 2026-08-29 (CLUS-54).** This section was written on 2026-08-16 and
listed E0 and E1 as entirely absent. QR-136 shipped them on 2026-08-20
(countersigned) and this section was not updated, so for nine days it asserted
the opposite of the truth. The obsolete list is preserved below the line as a
historical record; **the live list is this one.**

**As of 2026-08-29 — still not implemented:**

- **E2 and everything below the cut line.** Per-resource keys (§3.4), key
  epochs (§3.6), rekey-on-revoke (§9), forward secrecy / key transparency /
  E2E account recovery (E4). Specified or deferred, none built.
- **No key revocation.** §9 is deferred to 2028. Session revocation
  (`DSessionRevoked`) exists on the wire and is a different thing: it ends a
  session, it does not rotate or invalidate a key.
- **No capability enforcement.** §8's seam is specified; the capability core is
  not implemented, and grants are still household-wide prefix strings from an
  environment variable.
- **The bootstrap model.** `api/src/device0.rs` implements the superseded
  server-side-keygen model (household keypair generated server-side, private
  seed persisted as `device0_seed_hex`). §4's root-of-authority /
  root-of-custody split is the target, not the current state. *(Carried
  forward from the 2026-08-16 text and not re-verified in this doc pass —
  check `api` before relying on it either way.)*
- **Not everything on the fabric is encrypted.** E1 is available and shipping,
  but payloads published on paths that do not call the encrypting API still
  transit and are retained in plaintext. §11 is the honest account of that gap;
  "E1 exists" is not "E1 is universal."
- **E0 is not yet byte-conformant.** `fabric-kit`'s E0 path works end to end but
  does not yet match §7's byte layout exactly; QR-228 / QR-229 close it. This
  document is normative.

**What may be said publicly, and when,** is §2.4. E1 having shipped narrows but
does not erase that discipline: a present-tense end-to-end-encryption claim is
now defensible **only** for the specific paths that actually encrypt, and a
blanket "Hearth is end-to-end encrypted" is still ahead of the code while the
previous bullet is open. Check §2.4 before writing marketing copy.

---

<details>
<summary><strong>Superseded — the 2026-08-16 text, kept as a record</strong></summary>

> **As of 2026-08-16:**
>
> - **No content encryption exists.** Every payload on the fabric is plaintext.
>   The hub reads all of it and retains it for seven days.
> - **No X25519 key exists anywhere in the stack.** `hub/pairing.rs` has reserved
>   storage (`set_device_sealing_key` / `device_sealing_key`) and nothing calls
>   it. No client and no gateway mints a sealing keypair.
> - **No HPKE, no AEAD, no KDF.** `ed25519-dalek` is the only cryptographic
>   dependency in `fabric-kit`.
> - **No attestation chain is verified.** F6 attestations are persisted (commit
>   `8c63496`) and read back; nothing checks a chain of them, and F7 (§5.3) is not
>   filed against the wire yet.
> - **No roster-read wire path is implemented.** The shape is now ruled (§5.4:
>   `TRoster`/`RRoster`, envelope 50/51), but nothing implements it, and today a
>   device still cannot learn a peer's public key at all. Two record-side gaps
>   block implementation (§5.4.7): the persisted attestation on `hub` `main` has
>   no version field, and the reserved sealing-key storage cannot hold a
>   verifiable key.
> - **No capability enforcement.** Grants are household-wide prefix strings from
>   an environment variable.
> - **No revocation of anything.**

Every bullet above except the capability and revocation ones was overtaken by
QR-136 on 2026-08-20: X25519 sealing keys, RFC 9180 HPKE, XChaCha20-Poly1305
content encryption, F7 attestation verification, and the 7-step roster verify
path are all live and were live-verified end to end.

</details>

---

## 13. Open questions

Each is routed. None of them blocks a developer from starting on §§3, 5, 6, or 7.

| # | Question | Owner | Default if unanswered |
|---|---|---|---|
| ~~**Q1**~~ | ~~**D3 — the roster-read wire path.**~~ **ANSWERED 2026-08-17** (messaging-architect): `TRoster`/`RRoster` at envelope 50/51, hub as untrusted directory — §5.4. Both record-side prerequisites this originally named are already resolved on `hub` `main` (§5.4.7) — implementable now, just not yet built. | — | — |
| **Q2** | **F7 — extend the pairing attestation to cover the sealing key** (§5.3). Two additive fields on `DPairClaim`. | **messaging-architect**, urgently | None. One-way door; every pairing meanwhile is permanently uncovered. |
| **Q3** | ~~**D4 — is `payload_type` plaintext or wrapped** (§6.5)?~~ **RULED 2026-08-18.** | **messaging-architect** | **Wrapped, under `qr.seal/box`** — see §6.5. On sealed topics the outer `payload_type` MUST be the constant `qr.seal/box`; the real type is sealed inside QRE1. |
| **Q4** | **Record shapes** — content-key holder set, epoch record, sealing key and binding on the roster record, root-holder set. | **data-architect.** The seam holds: they own what these *are*, this document owns what they *permit*. | — |
| **Q5** | **Migrate or re-provision the alpha population** (§11.5). | **DJ** — product trade, not a crypto question | Re-provision. |
| **Q6** | **The delegated-admission second tap** (§5.6b) — what the confirmation dialog says and where it lives. | **DJ + ux-engineer** | Explicit confirmation naming the requesting device and showing its fingerprint. |
| **Q7** | **Does the product state the total-loss trade at the choice point** (§4.6)? | **DJ** | Yes, at the choice point, in plain words, not in a footnote. |
| **Q8** | **Fingerprint legibility** (§5.5) — the current rendering fails contrast. | **ux-engineer** | WCAG AA minimum. This is a security requirement. |
| **Q9** | **Households with no gateway.** A household that self-hosts nothing and declines a hosted presence has no gateway. What holds the authority root, and what is always-online enough to answer a §5.6 key request? | **DJ + data-architect** | The user's primary device holds the root; §5.6 requests wait in catch-up. Works, but the UX of "your other phone must be on" needs an answer. |
| **Q10** | **Proof obligations** — see below. | **qa-engineer** | — |

### What must be proven, not asserted

For `qa-engineer`. These are the tests that distinguish a control that exists
from a control that is described.

1. **No plaintext household content appears in any outbound published frame once
   E1 is on** — verified mechanically, by scanning every payload the gateway
   publishes during a live run, not by inspection.
2. **A frame captured from topic A cannot be decrypted when replayed onto topic
   B** (§6.3's AAD binding).
3. **A frame cannot be re-published under a different `sender_device_id`** and
   still decrypt (§6.3).
4. **Catch-up across an epoch boundary decrypts; catch-up naming an expired
   epoch fails closed** rather than silently returning garbage or an empty
   message (§3.6, §6.6).
5. **A `ContentKeyGrant` addressed to device A is rejected when presented to
   device B** (§5.8 step 4).
6. **A `ContentKeyGrant` from a grantor whose attestation chain does not verify
   is rejected** (§5.8 step 3) — and specifically is not accepted on the hub's
   assertion alone.
7. **A device in `KEY_PENDING` publishes nothing** — no plaintext fallback under
   any code path (§5.6).
8. **A sealed blob cannot be opened by any device other than the target**,
   including another device on the same household (§7).
9. **A sealed blob past `not_after_ms` is rejected**, and a replayed `seal_id`
   is rejected (§7.3).
10. **A capability signed by the pre-handoff authority root is denied after
    rotation** (§4.3).
11. **No content key is ever wrapped to device-0's roster entry** after its death
    (§4.4, §11.3 step 3) — this is the silent-failure case in the migration and
    it needs a dedicated test.
12. **A household with zero root holders cannot be created by a stalled
    bootstrap** (§4.3).
13. **A principal holding approve-but-not-content-key cannot cause a content key
    to be wrapped to a new device** (§4.5).

**Roster read (§5.4).** Six more, and every one of them requires *mutating a
served reply* rather than asserting the happy path:

14. **A hub that substitutes `ed25519_pubkey` or `x25519_pubkey` in a
    `RosterEntry` fails verification** — tested by actually altering the bytes
    a test hub serves, not by reasoning about §5.4.2.
15. **An entry whose attestation is v1 is never wrapped to** (§5.4.5 step 7).
16. **An entry carrying a sealing key with no `sealing_binding_sig`, or with a
    `sealing_created_at_ms` that differs from the one inside the signature, is
    rejected** — not partially accepted (§5.4.1).
17. **A `truncated = true` reply is never used to conclude a device is absent**
    (§5.4.4) — the test is a wrap decision taken against a deliberately
    truncated roster.
18. **An attestation minted in household X and replayed into household Y's
    roster fails**, because its approver does not chain to Y's root (§5.4.3).
19. **A roster read across the device-0 handoff verifies via the transition
    attestation**, and a chain rooted only in the old root after
    `signed_at_ms` is rejected (§4.3, §5.4.5 step 1).

---

## Appendix A — signature and HPKE context strings

Every signed or KDF-bound string defined by this document, in one place. **All
are ASCII, all are followed by a `0x00` terminator, and all variable-length
fields inside the material are length-prefixed.** Adding a new signed structure
means adding a new context string here first.

| Context string | Used in | Signed/bound material |
|---|---|---|
| `qr-sealing-binding-v1` | §3.2 | device_id ‖ x25519_pubkey ‖ created_at_ms |
| `qr-pair-attest-v2` | §5.3 (F7) | device_id ‖ ed25519_pubkey ‖ x25519_pubkey ‖ signed_at_ms |
| `qr-authority-transition-v1` | §4.3 | account_id ‖ old_root_pk ‖ new_root_pk ‖ signed_at_ms |
| `qr-contentkey-v1` | §5.7 (HPKE `info`) | key_id ‖ epoch ‖ recipient_device_id |
| `qr-contentkey-grant-v1` | §5.7 (signature) | grant bytes ‖ recipient_device_id ‖ recipient_x25519_pubkey |
| `qr-seal-v1` | §7.2 (HPKE `info`) | seal_id ‖ recipient_device_id |
| `qr-seal-v1-sig` | §7.2 (signature) | blob bytes ‖ recipient_device_id ‖ recipient_x25519_pubkey |
| `qr-device-fingerprint-v1` | §5.5 | ed25519_pubkey ‖ x25519_pubkey |

**Pre-existing gap, recorded here rather than left implicit:** the F6 v1 pairing
attestation signs a bare 32-byte `new_device_pubkey` with **no context string
and no length prefix**. It is not exploitable today because no other 32-byte
Ed25519 signature material exists in the system, but it becomes a real
cross-protocol hazard the moment a second one does. F7 (§5.3) is the fix, and it
is one more reason to land F7 rather than defer it.

---

## Appendix B — magic numbers and wire discriminators

| Magic | Meaning | Section |
|---|---|---|
| `QRE1` (`51 52 45 31`) | E1 content ciphertext | §6.1 |
| `QRK1` (`51 52 4B 31`) | content-key grant | §5.7 |
| `QRS1` (`51 52 53 31`) | sealed blob (E0) | §7.2 |

| `suite_id` | Content AEAD | Section |
|---|---|---|
| `0x01` | XChaCha20-Poly1305 | §6.2 |

| `hpke_suite` | KEM / KDF / AEAD / mode | Section |
|---|---|---|
| `0x01` | DHKEM(X25519, HKDF-SHA256) `0x0020` / HKDF-SHA256 `0x0001` / ChaCha20-Poly1305 `0x0003` / `mode_base` | §5.7 |

| Envelope field | Message | Section |
|---|---|---|
| `50` | `TRoster` | §5.4.1 |
| `51` | `RRoster` | §5.4.1 |
| `52`–`59` | **reserved**, roster family | §5.4.1 |
| `106` | **reserved**, undefined — a future `DRosterChanged` | §5.4.6 item 6 |

| `ErrorCode` | Meaning | Section |
|---|---|---|
| `50` | `ROSTER_RATE_LIMITED` | §5.4.1 |
| `51`–`59` | **reserved**, roster family | §5.4.1 |

**Normative:** an unrecognized magic, version, `suite_id`, `hpke_suite`,
attestation version, or a set reserved flag bit MUST deny. There is no
permissive parse, no best-effort decrypt, and no downgrade path in any
direction.

**Note on Appendix A:** the roster read (§5.4) introduces **no new context
string**. It transports signatures defined elsewhere in this document —
`qr-pair-attest-v2` (§5.3), `qr-sealing-binding-v1` (§3.2), and
`qr-authority-transition-v1` (§4.3) — and defines none of its own. A roster
read that needs a new signature is a roster read that has grown a trust
decision it should not have.
