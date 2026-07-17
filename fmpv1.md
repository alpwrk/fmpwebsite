<!--
fmp — Federated Music Protocol
Whitepaper / Design Documentation
Copyright (c) 2026 alpwrk
Licensed under CC BY-SA 4.0. See https://creativecommons.org/licenses/by-sa/4.0/
-->

# fmp — Federated Music Protocol

**A decentralized, music-native protocol for publishing, federating, streaming, and downloading music — without DRM, without server lock-in, without consensus machinery.**

Version: Whitepaper v1.0
Author: alpwrk · 2026
License (this document): CC BY-SA 4.0

---

## Abstract

fmp is a protocol for distributing music across independent, self-hosted servers. Identity is a cryptographic key rather than a server account; tracks are content-addressed and provenance-signed; audio stays at its origin while only metadata federates on demand. Hard distributed-systems problems (global consensus, Sybil resistance, unique naming) are avoided by construction rather than solved with heavy machinery, so the network survives partitions by design and stays operationally lean enough to run on a homelab.

This document has two parts. **Part I — Vision** explains what fmp is, why it exists, and the design principles behind it. **Part II — Technical Appendix** specifies the concrete mechanisms: the binary wire protocol over TCP/TLS, the packet model, the federation and caching algorithms, and a sketch of the signed download container format.

Sections marked **[OPEN]** are unresolved design decisions where this document proposes a default but does not yet commit. They are the natural next points of discussion before a reference implementation is frozen.

---

# Part I — Vision

## 1. Motivation

Music distribution today sits on rented ground. Streaming platforms hold the catalog, the identity, and the reach; the listener owns nothing and the uploader controls nothing once uploaded. DRM turns purchased music into a revocable license. Geo-blocking ("in deinem Land nicht verfügbar") partitions the catalog by jurisdiction. Server accounts tie identity to a single provider, so leaving means starting over.

fmp starts from a different premise: **you own what you publish, you own what you download, and no single server owns the network.** It is a FOSS/suckless answer to platform lock-in — the same argument that applies to proprietary software and to hardware right-to-repair, applied to music distribution.

## 2. What fmp is (and is not)

fmp **is** a wire protocol plus a data model. It defines how independent servers describe tracks, how clients discover and fetch them, how identity works, and how servers federate metadata on demand.

fmp **is not** a platform, a company, a client, or a catalog. There is no central registry, no canonical server, and no protocol-level authority. Anyone can run a server; anyone can write a client; the protocol is the only thing in common.

### Honest positioning

fmp is **not** the first federated system with a free client. Nostr, the ActivityPub Fediverse, and Bluesky already operate at scale, and Nostr in particular matches the key-is-identity, free-client model closely. What is distinct about fmp is that it is **music-native by design**:

- a popularity-based caching/replication economy tuned for large audio blobs,
- an owned, DJ-compatible, signed download format,
- trust deliberately kept out of the protocol,
- a lean Rust binary-protocol philosophy rather than generic JSON-over-WebSocket.

fmp does not aim for mass adoption. It aims to be a well-built niche tool that succeeds if onboarding is trivial (a single `docker compose up`) and the protocol stays small enough to reimplement.

## 3. Core principles

- **Key is identity.** An Ed25519 keypair on the client is the account. No registration, no password recovery, no homeserver lock-in. Losing the key means losing the identity — this is a deliberate trade, not an oversight.
- **Own what you publish.** Downloads are DRM-free, real, playable files in an open, signed format.
- **Provenance, not protection.** Every track is cryptographically bound to its uploader's key. This proves origin; it does not prevent copying. fmp signs, it does not lock.
- **No consensus.** Every node acts on local information only. There is no global agreement to reach, so there is nothing to partition.
- **Instance vs. client sovereignty.** Instances decide what they carry; clients decide what they reach. Neither overrides the other. Defederation by an instance is never censorship of the network, because content-addressed tracks can be routed around it by the client.
- **Popularity-based caching.** Content replicates toward demand. Origin servers are protected from raw leeching because popular content is cached closer to where it is requested.
- **Trust is not in the protocol.** There is no protocol-level web-of-trust. This removes the Sybil problem at the protocol layer entirely: since no protocol decision depends on "how trusted" a key is, creating many keys buys an attacker nothing at the protocol level. Trust, where wanted, lives in clients and instances, not in fmp.

## 4. Identity model

Identity is an **Ed25519 keypair**. The public key *is* the user. There are no usernames owned at the protocol level.

A **handle** is a best-effort, cosmetic label — a display convenience, not a unique identifier. Guaranteed globally unique names would require either a central naming authority or a blockchain, both of which fmp rejects. So handles collide freely, and the key is always the disambiguator. Clients may show `handle@fingerprint` or similar, but the fingerprint is authoritative.

## 5. Track model

A track is identified by a **triplet**:

```
(name, content_hash, uploader_pubkey)
```

- **name** — human-readable, non-unique, cosmetic.
- **content_hash** — Blake3 hash of the audio bytes. This is the content-address: identical audio has an identical hash regardless of who uploaded it or what it is called.
- **uploader_pubkey** — the Ed25519 public key that published this particular listing.

Because the hash is content-derived, the same recording uploaded by two people produces the same `content_hash` under two different pubkeys — deduplication and cross-origin routing fall out naturally, while provenance stays attributable per uploader.

## 6. Federation model — lazy, not eager

fmp federates **metadata lazily**. Servers do **not** eagerly replicate each other's full catalogs (the model fmp explicitly rejects as overengineering). Instead:

- Each server holds its own tracks and metadata locally.
- Federation happens **on demand**: when a local user actively reaches toward another server (searching it, following a key hosted there, fetching a track), the servers exchange the relevant metadata for that interaction — analogous to how DNS resolves on demand rather than mirroring the whole namespace.
- **Audio never eagerly federates.** Audio bytes stay at their origin server. Only metadata crosses server boundaries, and only when needed. Audio moves solely through the popularity-based cache (Section 7), pulled toward demand rather than pushed everywhere.

This keeps small servers small: running an fmp instance does not mean hosting the whole network's catalog.

## 7. Caching & replication economy

Because audio stays at origin, a naive network would hammer popular origins into the ground. fmp solves this with a **popularity-based cache** that pulls hot content closer to demand:

- **Scoring: LFU-with-aging.** Each cached item has a score that increases with access frequency and decays over time, so yesterday's viral track eventually yields cache space to today's without being pinned forever.
- **Eviction: high/low watermark.** When cache usage crosses a **high watermark**, the server evicts lowest-scoring items until it falls back to a **low watermark**. This gives hysteresis — eviction runs in batches rather than thrashing item-by-item at the boundary.
- **Pinning.** An operator can pin items to exempt them from eviction (e.g. a server's own uploads, or content it wants to guarantee availability for).

The effect is an emergent CDN: popular tracks replicate outward toward the servers whose users request them, protecting origins from leeching, while unpopular tracks stay only at origin and cost the network nothing.

## 8. Moderation & sovereignty

fmp has **no protocol-level moderation** — there is nothing to moderate centrally because there is no center. Moderation is entirely local, at three levels:

1. **Pubkey blacklist** — an instance refuses to carry or serve content from specific keys.
2. **Defederation** — an instance refuses to interact with another specific instance.
3. **Client routing** — a client chooses which instances it reaches through, and can route around any instance.

The key property: **defederation is never censorship of the network.** Because tracks are content-addressed, a client blocked from one route to a track can reach the same `content_hash` through another instance that carries it. Instances control their own walls; clients control their own paths; neither can unilaterally remove content from the network.

## 9. Governance & versioning

Protocol evolution uses **soft governance**:

- Migration between protocol versions is **voluntary**.
- **Hard cutoff dates** for deprecated versions are agreed between maintainers and contributors, not imposed centrally.
- There is no on-chain or consensus-based versioning; coordination is social, documented, and scheduled.

This matches fmp's overall stance: coordination where it genuinely helps interop, freedom everywhere else.

## 10. Licensing intent

- **Specification & documentation** (this whitepaper, `PROTOCOL.md`, the typeset PDF): **CC BY-SA 4.0** — share and adapt with attribution, keep derivatives under the same license.
- **Reference implementation** (when it lands): licensed separately. Intended **AGPLv3** for server software (so offering-as-a-service stays open) and possibly **MIT** for parts where maximal reuse is wanted. The exact split will be stated when code lands.

---

# Part II — Technical Appendix

> This appendix specifies concrete mechanisms. Where the earlier specification left a decision open, a default is proposed and flagged **[OPEN]**.

## A. Transport & framing

fmp runs a **custom lean binary protocol over TCP with mandatory TLS** (TLS 1.3). No plaintext transport exists. There is no HTTP layer; the binary framing below is spoken directly over the TLS stream.

Rationale for a binary protocol rather than JSON-over-WebSocket: audio metadata and control traffic are high-volume and well-structured, and a fixed binary framing keeps parsing cheap and the wire small — consistent with the lean/Rust philosophy.

### A.1 Frame layout

Every message is a length-prefixed frame:

```
┌──────────┬──────────┬───────────┬──────────────┬─────────────┐
│  magic   │ version  │  pkt_type │  body_len    │    body     │
│  2 bytes │  1 byte  │  2 bytes  │   4 bytes    │  body_len B │
└──────────┴──────────┴───────────┴──────────────┴─────────────┘
```

- **magic** (`0xFM`, i.e. `0x46 0x4D`) — quick frame-sync / sanity check.
- **version** — protocol major version; mismatched majors are rejected per Section 9 governance.
- **pkt_type** — 16-bit packet type identifier (see A.2). Unknown non-mandatory types are **silently ignored** (forward-compatibility: a client/server may skip a frame it doesn't understand by using `body_len`).
- **body_len** — unsigned 32-bit big-endian length of the body.
- **body** — packet-type-specific payload. **[OPEN]** body encoding: proposed default is a compact binary encoding (length-prefixed fields, fixed-width integers big-endian). Candidate concrete choices: hand-rolled, MessagePack, or a schema format (bincode/protobuf-lite). Recommendation: **bincode-style fixed encoding** for the reference impl since both ends are Rust, revisit if a non-Rust client appears.

### A.2 Packet namespace

Packet types are grouped by prefix, so unknown *optional* groups can be dropped while mandatory ones are enforced. **[OPEN]** exact numeric assignments — the grouping below is the model, concrete IDs to be fixed in `PROTOCOL.md`.

| Group      | Purpose                                              | Mandatory? |
|------------|------------------------------------------------------|------------|
| `sys.*`    | Handshake, capability negotiation, version, ping     | Yes        |
| `id.*`     | Key exchange, challenge/response auth, handle claim   | Yes        |
| `track.*`  | Track listing, metadata query, content_hash lookup    | Yes        |
| `fed.*`    | Lazy federation: on-demand metadata pull between servers | Yes (server-server) |
| `cache.*`  | Cache negotiation, replication pull, pin hints        | Optional   |
| `stream.*` | Audio streaming / chunked fetch                       | Optional   |
| `dl.*`     | Signed download container transfer (see D)            | Optional   |

Unknown packets in an **optional** group are silently ignored (per A.1). Unknown packets in a **mandatory** group are a protocol error and close the connection.

### A.3 Capability negotiation

On connect, both peers exchange a `sys.caps` packet listing the packet groups they support. Each side then filters outbound optional packets to what the peer advertised, so unsupported optional traffic is never sent — saving bandwidth and avoiding silent drops on the far side. Mandatory groups are assumed and not negotiated.

## B. Identity & authentication on the wire

- **Keys:** Ed25519. The public key is the identity.
- **Auth:** challenge/response. On connect, after `sys.caps`, a server issues an `id.challenge` (random nonce); the client returns `id.proof` = Ed25519 signature over the nonce (plus a context string binding it to this session/TLS channel to prevent replay). No password, no shared secret, no account record required — possession of the private key *is* authentication.
- **Handles:** an `id.handle` packet carries a cosmetic label. It is **not** verified for uniqueness by the protocol; collisions are expected and resolved by fingerprint on the client side.

**[OPEN]** channel binding detail: whether the signed context includes the TLS exporter value (RFC 5705 keying material exporter) or a simpler session nonce. Recommendation: **TLS exporter binding**, so an `id.proof` cannot be replayed onto a different TLS channel.

## C. Track metadata, federation & caching packets

### C.1 Track representation

A `track.meta` body carries at minimum:

```
name            : length-prefixed UTF-8   (cosmetic, non-unique)
content_hash    : 32 bytes                (Blake3-256 of audio bytes)
uploader_pubkey : 32 bytes                (Ed25519)
uploader_sig    : 64 bytes                (Ed25519 signature over the metadata)
size            : u64                     (audio byte length)
codec           : u8 enum                 (see [OPEN] below)
duration_ms     : u32
created_at      : u64                     (unix seconds, informational only)
```

The `uploader_sig` binds `(name, content_hash, size, codec, duration_ms, created_at)` to the uploader key — this is the provenance signature. **[OPEN]** codec enum set: proposed initial set `{flac, mp3, opus, wav, aac}`, extensible; a client that doesn't support a codec still sees valid metadata and can route/skip.

### C.2 Lazy federation (`fed.*`)

Server-to-server metadata pull is on-demand:

- `fed.query` — "do you have metadata matching X?" where X is a content_hash, an uploader_pubkey, or a search term. **[OPEN]** search semantics: the protocol can guarantee exact `content_hash` and `pubkey` lookup cheaply; fuzzy text search across servers is expensive and arguably out of protocol scope. Recommendation: **protocol guarantees exact-match federation only** (`content_hash`, `pubkey`); text search is a local, per-instance concern over that instance's own index, not a federated query.
- `fed.meta` — response carrying zero or more `track.meta` bodies.
- No standing subscription, no push. Each federation exchange is scoped to the interaction that triggered it, then done.

### C.3 Caching (`cache.*`)

- `cache.pull` — a server requests audio bytes for a `content_hash` from an origin or from another cache that advertised it.
- `cache.have` — advertises that this server currently caches a given `content_hash` (used so clients/servers can find a nearer copy than origin).
- `cache.pin_hint` — **[OPEN]** whether pinning is purely a local operator action (recommended) or whether an origin can *request* that peers pin its content. Recommendation: **pinning is local-only**; no server can compel another to pin, consistent with instance sovereignty.

Scoring (LFU-with-aging) and watermark eviction are **local implementation behavior**, not wire-visible — the protocol only carries `pull`/`have`, and each server runs its own cache policy. This keeps the protocol small and lets operators tune caches independently.

## D. Signed download container format (sketch)

The "own what you download, DJ-compatible" format. Goal: a single file that carries the audio **plus** verifiable provenance, playable in normal tools while remaining checkable.

**[OPEN]** the entire container is a sketch, not frozen. Two viable directions:

- **D-1 (sidecar):** ship the raw audio file **unmodified** plus a small detached `.fmpsig` metadata+signature file. Pro: audio is byte-identical to a normal file, plays everywhere, DJ software is happy, no muxing. Con: two files to keep together; provenance is lost if the sidecar is dropped.
- **D-2 (embedded container):** wrap audio in a container with a signed header block. Pro: single self-verifying file. Con: needs either a custom container (tooling burden) or careful use of existing metadata frames.

**Recommendation:** **D-1 sidecar as the primary format** (maximally compatible, suckless, no muxing, the audio stays a real playable file exactly as principle §2 demands), with an optional D-2 for users who want a single self-contained artifact.

### D.1 Sidecar layout (recommended)

```
mytrack.flac            ← untouched audio, plays in anything
mytrack.flac.fmpsig     ← detached provenance sidecar
```

`.fmpsig` binary layout:

```
┌────────────┬─────────┬──────────────────────────────────────────┐
│  field     │  size   │  meaning                                   │
├────────────┼─────────┼──────────────────────────────────────────┤
│ magic      │ 4 B     │ "FSIG"                                      │
│ version    │ 1 B     │ sidecar format version                      │
│ content_hash│ 32 B   │ Blake3-256 of the audio file it accompanies │
│ pubkey     │ 32 B    │ Ed25519 uploader key                        │
│ name_len   │ 2 B     │ length of name field                        │
│ name       │ var     │ UTF-8 track name                            │
│ codec      │ 1 B     │ codec enum (matches C.1)                    │
│ duration   │ 4 B     │ ms                                          │
│ created_at │ 8 B     │ unix seconds                                │
│ signature  │ 64 B    │ Ed25519 over all preceding fields           │
└────────────┴─────────┴──────────────────────────────────────────┘
```

Verification: recompute Blake3 over the audio file, check it equals `content_hash`, then verify `signature` over the header using `pubkey`. If both pass, the file's origin and integrity are proven. Note this proves **who published it and that it is unmodified** — it deliberately does **not** prevent copying (provenance, not protection).

### D.2 Embedded container (optional)

If a single self-verifying file is wanted, the same header block is prepended to (or embedded as a leading metadata chunk in) the audio, and the audio payload follows. **[OPEN]** exact muxing strategy and which codecs' native metadata frames to reuse vs. a wrapping container. Deferred until there is demand — the sidecar covers the core need.

## E. Reference implementation plan

- **Language/stack:** Rust — `tokio` (async runtime), `axum` **[OPEN]** (axum is HTTP-oriented; since the wire protocol is raw binary-over-TLS, the reference impl likely uses `tokio` TCP + `rustls` directly, with `axum` only if/where an auxiliary HTTP surface is wanted such as a health endpoint or web UI). `sqlx` + SQLite for local metadata/index storage.
- **Crypto:** `ed25519-dalek` (identity/signatures), `blake3` (content hashing), `rustls` (TLS 1.3).
- **First milestone (recommended entry point):** the **identity layer** — Ed25519 keygen, the `id.challenge`/`id.proof` handshake, and `content_hash` computation. It is a self-contained, testable brick that everything else builds on.
- **Onboarding target:** single `docker compose up` to stand up an instance — trivial onboarding is the adoption lever for a niche FOSS protocol.

---

## Appendix Z — Open questions summary

The **[OPEN]** items above, collected for the next design pass:

1. Body encoding on the wire (recommend bincode-style fixed binary; revisit for non-Rust clients).
2. Concrete numeric packet-type assignments.
3. Channel binding for `id.proof` (recommend TLS exporter binding).
4. Codec enum set (recommend `{flac, mp3, opus, wav, aac}`, extensible).
5. Federated search semantics (recommend exact-match-only federation; text search stays local).
6. Pinning authority (recommend local-only, no remote compel).
7. Download container direction (recommend D-1 sidecar primary, D-2 embedded optional).
8. Role of `axum` given a raw binary transport (recommend tokio+rustls direct; axum only for auxiliary HTTP).

---

*fmp — Federated Music Protocol · Whitepaper v1.0 · alpwrk · 2026 · CC BY-SA 4.0*
