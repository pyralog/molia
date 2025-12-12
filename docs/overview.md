# Molia DHT

A practical, production-ready distributed hash table design with security, privacy, performance, and operability baked in. This blueprint targets Internet-scale deployment across desktops, servers, mobile, and browsers.

---

## 0) Goals & Non‑Goals

**Goals**
- Sub‑second lookups at Internet scale; median ≤300 ms, P95 ≤1 s.
- Robust under high churn and partial partitions.
- First‑class NAT traversal and browser support.
- Strong data integrity, authenticated mutability, and policy‑driven retention.
- Privacy: query blinding + minimization; optional anonymity for lookups.
- Observability + upgradeability: rolling protocol upgrades without flag days.

**Non‑Goals**
- Global consensus (use a separate ledger if needed).
- Strong anonymity against global adversaries (we aim for practical privacy, not Tor‑grade anonymity).

---

## 1) Overlay: ID Space, Distance, and Topology

- **Node IDs**: 256‑bit (Blake3 of the node’s static Ed25519 public key). Uniform, self‑certifying.
- **Key Space**: 256‑bit content and namespace keys (multihash‑style). Prefixes enable range/prefix features.
- **Distance Metric**: XOR distance (Kademlia‑style) with **latency bias**: tie‑breaks by observed RTT.
- **Routing Table**: k‑bucket tree with adaptive split; **K = 16…32** per bucket depending on memory tier.
  - **Bucket Aging**: LRU within bucket; periodically ping oldest; evict unresponsive peers.
  - **Proximity Neighbor Selection (PNS)**: prefer lower RTT peers for same distance.
- **Routing Mode**: Iterative by default (caller queries peers in parallel), with optional recursive for low‑latency clusters.
- **Concurrency (α)**: Adaptive **α = 3..8**; scales up on packet loss/slow paths.

---

## 2) Transport & NAT Traversal

- **Primary Transport**: **UDP** with WireGuard tunnels (userspace).
- **Fallbacks**: TCP (hole‑punching where possible); **WebRTC DataChannels** for browsers.
- **NAT/Firewall**: STUN‑like discovery via lightweight **rendezvous relays**; TURN‑like relaying as last resort.
- **Hole Punching**: Automated coordination using signed peer records; consent‑freshness ≤30 min.
- **Address Records**: Multiaddr‑style; include observed addresses and proof‑of‑reachability.

---

## 3) Record Model (Immutable, Mutable, Provider, and Index)

All records share a common envelope:
```
message Record {
  bytes key;                  // 32 bytes
  bytes value;                // app-defined payload
  uint64 sequence;            // for mutable; 0 for immutable
  uint64 ttl_secs;            // soft TTL
  uint64 not_before_unix;     // optional delay publication
  bytes owner_pubkey;         // Ed25519 (mutable/provider)
  bytes signature;            // sig over (key,value,sequence,ttl,nb)
  bytes validators;           // bitmap of passed validations
}
```

**Types**
- **Immutable**: `key = hash(value)`; stored as content‑addressed blobs.
- **Mutable (NAMED)**: `key = hash(owner_pubkey || namespace || salt)`, monotonic `sequence` with sig.
- **Provider (Routing)**: maps a content key → list of provider peer IDs + metadata (chunking supported).
- **Secondary Index** (optional): prefix‑tree shards to support prefix/range queries via bounded walk.

**Validation**
- Immutable: hash check.
- Mutable: Ed25519 signature, monotonic sequence, optional schema (CID, protobuf, JSON schema).
- Provider: signature from provider; **rate‑limited per peer** to mitigate spam.

---

## 4) Replication, Caching, and Availability

- **Replica Set**: store on the **k‑closest** nodes to `key`, default **k=20**.
- **Redundancy**: **Erasure coding** (Reed–Solomon): `data:parity = 6:3` or `10:4` for large objects.
- **Opportunistic Caching**: any node on a successful path may cache with a small TTL (e.g., 1–10 min) + Bloom filter hints to avoid over‑replication.
- **Refresh**: owner (or stewards) republish mutable/provider records at **TTL/2**; passive refresh when queried.
- **Garbage Collection**: expiry by TTL; **tombstones** for rapid deletion of mutable keys.

---

## 5) Lookup Algorithm (Iterative, Privacy‑Hardened)

1. Start with the **α closest known peers** to `target` from the local table.
2. Query in parallel with **request shaping** (token bucket per peer + per /24).
3. On responses, merge and sort by XOR distance, preferring lower RTT.
4. Repeat until no closer peers are found; then query final k for the record.
5. **Privacy add‑ons** (configurable):
   - **Query Blinding**: send padded probes for neighbors of `target` (± small prefix delta) to mask the exact key.
   - **Two‑hop Relay**: route the first leg via a randomly chosen relay from a disjoint bucket.
   - **Dandelion‑like stem/flare** for provider announcements.

**Complexity**: `O(log N)` hops; expected hops 3–5 at Internet scale.

---

## 6) Security, Abuse Resistance, and Trust

- **Identity**: Ed25519 keypairs; NodeID = BLAKE3(pubkey). **Self‑certifying peer IDs**.
- **Transport Security**: WireGuard protocol (no tunneling, just state machine; e.g., [BoringTun](https://github.com/cloudflare/boringtun)) with identity bound to NodeID.
- **Silent by default**: unknown peers receive no reply prior to a valid WireGuard handshake (cookie gating).
- **Sybil Resistance** (mix‑and‑match based on threat):
  - **Capped Admission**: proof‑of‑work (adaptive), or proof‑of‑uptime, or proof‑of‑resource (storage bw).
  - **DHT‑Placement Hardening**: salt bucket boundaries daily; discourage targeted collocation.
  - **Behavioral Scoring**: EWMA of responsiveness, correctness, equivocation; quarantine bad actors.
- **Rate Limiting**: per peer/IP/token bucket; **challenge‑response** puzzles during spikes.
- **DoS Defenses**: bounded request sizes, early abort on invalid envelopes, moving target ports for relays.
- **Privacy**: minimize metadata; optional **oblivious fetch** via relay pools; encrypt mutable payloads end‑to‑end.

---

## 7) Wire Protocol (RPCs over UDP)

**Core RPCs**
- `FIND_NODE(target_id)` → peers[]
- `FIND_VALUE(key)` → {record | providers[] | closer_peers[]}
- `STORE(record)` → {accepted, reason}
- `ANNOUNCE_PROVIDER(key, self_descriptor)`
- `PING()` / `PONG()` with latency + load hints
- `NEGOTIATE(capabilities)` → agreed features, versions

**Negotiation & Upgrades**
- **Capabilities** bitmap (e.g., erasure, privacy, recursive, webRTC, vNext schemas).
- **Versioning**: semantic; nodes keep multiple handlers enabled; canary percentage rollout.

**Encoding**
- Protobuf; strict size limits.

---

## 8) Bootstrap & Membership

- **Bootstrap Sources**: Cache from last run, hardcoded well‑knowns, BitTorrent Mainline DHT, DNS seeds (multi‑A/AAAA), local discovery.
- **Rendezvous**: simple topic‑based discovery (hash(topic) bucket) for swarms and pub/sub overlays.
- **Liveness**: background table refresh (1 probe per bucket per 30–60 min) with jitter.

---

## 9) Operability & Observability

**Metrics** (Prometheus‑style)
- Lookup latency histograms, hop count distribution, success rate.
- Routing table health (peers per bucket, stale ratio), churn rate.
- NAT types mix; relay utilization; hole‑punch success.
- Store/fetch throughput, cache hit ratio, record sizes.

**Tracing**
- Optional distributed tracing headers per lookup (requester opt‑in), redacted by default.

**Logging**
- Structured logs with privacy scrubbers; rotating; sampling under load.

**Dashboards**
- SLOs and heatmaps by ASN/region.

---

## 10) Advanced Features

- **Prefix & Range Queries**: bounded skip‑graph overlay on top of XOR, keyed by prefixes; opt‑in.
- **Nearby Reads**: latency‑aware read‑nearest within the k‑set (fast path for geo‑distributed apps).
- **Content Pinning & Stewardship**: policy roles that keep specific keys alive with tighter SLOs.
- **Data Availability Sampling (DAS)**: lightweight probes on erasure‑coded chunks to ensure availability.
- **CRDT‑backed Mutable Sets**: grow‑only / add‑win‑map records with embedded anti‑entropy.
- **Pluggable Incentives**: tokenless credit (tit‑for‑tat buckets) or external rewards via side‑channels.
- **Mobile Mode**: duty‑cycled participation; push‑assisted wake for refresh/announce via relays.

---

## 11) API Sketch (Client Library)

```ts
interface DHT {
  get(key: Key, opts?: { privacy?: "none"|"blind"|"relay", timeoutMs?: number }): Promise<Value|NotFound>
  put(value: Value, opts?: { ttlSec?: number }): Promise<Hash>
  putMutable(ns: Namespace, value: Value, opts?: { seq?: number, ttlSec?: number }): Promise<{key: Key, seq: number}>
  findProviders(key: Key, max?: number): AsyncIterable<Provider>
  announceProvider(key: Key, meta?: ProviderMeta): Promise<void>
  joinBootstrap(seeds?: Addr[]): Promise<void>
}
```

---

## 12) Parameter Defaults (for MVP)

- `K=20`, `α=4`, `TTL=24h` for mutable/provider; immutable cached `5–30 min`.
- Erasure coding `10:4` for blobs ≥1 MiB; otherwise replicate.
- Relay budget ≤10% of egress; cap per‑peer ≤64 kbps sustained unless reputation allows burst.
- Privacy: **query blinding enabled** by default; 2‑hop relay off by default.

---

## 13) Implementation Roadmap (12 Weeks)

**Wk 1–2**: Core types, crypto (Ed25519, BLAKE3), UDP server/client, protobufs.

**Wk 3–4**: Routing table + iterative lookups; bootstrap; basic STORE/FIND_VALUE.

**Wk 5–6**: NAT traversal (rendezvous, STUN‑like), provider records, erasure coding.

**Wk 7–8**: Mutable records + signatures, TTL/refresh, opportunistic caching.

**Wk 9**: Privacy features (query blinding), rate limiting, abuse scoring.

**Wk 10**: Observability (metrics, logs, basic dashboards), chaos tests (churn/partitions).

**Wk 11**: Browser/WebRTC integration, relay budgeting, mobile mode.

**Wk 12**: Hardening, fuzzing, perf tuning, staged rollout.

---

## 14) Test Plan (Key Scenarios)

- **Churn Soak**: 30% nodes join/leave per minute; maintain ≥95% lookup success ≤1 s P95.
- **Partition**: simulate 3 regional islands; measure heal time and stale provider aging.
- **Adversarial**: 20% Sybils near a target prefix; verify placement hardening + rate limits.
- **NAT Mix**: full cone/symmetric; hole‑punch success ≥85% with relay fallback.
- **Data Durability**: random blob loss; ensure reconstruction via erasure + refresh.

---

## 15) Compliance & Policy Guardrails

- No PII in routing/provider records.
- Support **tombstones** and right‑to‑erasure for mutable keys; immutable content requires owner‑side key management.
- Configurable regional pinning (avoid storing in prohibited regions if operators opt in).

---

## 16) Build Notes

- Language suggestions: **Rust** (boringtun, ed25519‑dalek, blake3, prost).
- Keep hot paths lock‑free; use arenas for message buffers; avoid heap churn.
- Persistent peerstore with crash‑safe WAL.

---

## 17) Nice‑to‑Haves (v2)

- Private Information Retrieval (PIR) for high‑sensitivity lookups.
- Zero‑knowledge proofs of storage (for incentive layers).
- Gossip‑sub bridge for pub/sub applications.

---

## 18) Glossary

- **k‑bucket**: routing table bucket holding peers within a distance range.
- **PNS**: proximity neighbor selection.
- **Relay**: third‑party peer forwarding packets for NAT‑blocked peers.
- **Erasure Coding**: adds parity chunks so any `k` of `n` chunks reconstruct data.

---
