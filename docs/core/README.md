# Core Concepts

This directory contains documentation on fundamental DHT concepts and algorithms that underpin Molia.

## Documents

### [Kademlia Algorithm](kademlia.md)
Comprehensive guide to the Kademlia distributed hash table, including:
- XOR distance metric and its properties
- k-bucket routing table structure
- Iterative lookup algorithm (FIND_NODE, FIND_VALUE)
- RPCs: PING, STORE, FIND_NODE, FIND_VALUE
- Bucket maintenance (LRU, refresh, splitting)
- Worked examples and implementation tips

**Key Concepts**:
- **Distance**: `d(a,b) = a ⊕ b` (bitwise XOR)
- **k-buckets**: Exponential distance bands from your own ID
- **Parallelism (α)**: Query multiple peers concurrently for resilience
- **Replication**: Store on k-closest nodes to the key

**Why Kademlia?**
- Logarithmic lookup complexity: O(log N) hops
- Resilient to churn via LRU and parallelism
- Symmetric metric enables efficient routing
- Well-studied and battle-tested (BitTorrent, IPFS)

---

## Molia Enhancements

Molia builds on Kademlia with:
- **Latency-biased routing**: Prefer lower-RTT peers at equal XOR distance
- **Adaptive α**: Scale parallelism up on loss, down on healthy paths
- **Query blinding**: Optional privacy via neighbor probes
- **WireGuard security**: All traffic authenticated and encrypted
- **Sybil resistance**: Proof-of-work and behavioral scoring

See [../OVERVIEW.md](../OVERVIEW.md) for the complete architecture.

---

[← Back to Documentation](../)

