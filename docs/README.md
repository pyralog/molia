# Molia DHT Documentation

Welcome to the Molia DHT documentation. This directory contains comprehensive technical documentation organized by topic.

---

## Quick Navigation

### 📖 Start Here
- **[overview.md](overview.md)** - Complete architecture blueprint and design rationale

### 🏛️ [Architecture](architecture/)
- [Shared-Nothing Architecture](architecture/shared-nothing-architecture.md) - Per-core sharding and isolation
- [Zero-Allocation Design](architecture/zero-allocation-design.md) - Hot-path memory management

### 🔑 [Core Concepts](core/)
- [Kademlia Algorithm](core/kademlia.md) - DHT fundamentals and XOR routing

### 🌐 [Networking](networking/)
- [I/O Design](networking/io-design.md) - Event loops, buffers, batch I/O
- [Transport & NAT Traversal](networking/transport-nat-traversal.md) - UDP, hole punching, relays
- [Wire Protocol](networking/wire-protocol.md) - RPC messages and encoding

### 🔒 [Security](security/)
- [WireGuard Integration](security/wireguard-integration.md) - Transport security per shard
- [Sybil Resistance](security/sybil-resistance.md) - PoW, rate limiting, behavioral scoring

### ⚡ [Advanced](advanced/)
- [Linux eBPF Optimization](advanced/linux-reuseport-ebpf.md) - Kernel-space packet steering

---

## Documentation Map

```
docs/
├── overview.md                          # 🌟 Start here - complete blueprint
│
├── architecture/                        # Design patterns
│   ├── shared-nothing-architecture.md   # Isolation & sharding
│   └── zero-allocation-design.md        # Memory management
│
├── core/                                # DHT fundamentals
│   └── kademlia.md                      # XOR routing & k-buckets
│
├── networking/                          # Network stack
│   ├── io-design.md                     # I/O model
│   ├── transport-nat-traversal.md       # Connectivity
│   └── wire-protocol.md                 # RPCs & encoding
│
├── security/                            # Security & abuse resistance
│   ├── wireguard-integration.md         # Transport security
│   └── sybil-resistance.md              # Sybil defenses
│
└── advanced/                            # Platform-specific optimizations
    └── linux-reuseport-ebpf.md          # eBPF steering
```

---

## Reading Paths

### For New Contributors
1. [overview.md](overview.md) - Get the big picture
2. [core/kademlia.md](core/kademlia.md) - Understand the DHT algorithm
3. [architecture/shared-nothing-architecture.md](architecture/shared-nothing-architecture.md) - Learn the system structure
4. [networking/wire-protocol.md](networking/wire-protocol.md) - Study the protocol

### For Performance Engineers
1. [architecture/zero-allocation-design.md](architecture/zero-allocation-design.md)
2. [networking/io-design.md](networking/io-design.md)
3. [architecture/shared-nothing-architecture.md](architecture/shared-nothing-architecture.md)
4. [advanced/linux-reuseport-ebpf.md](advanced/linux-reuseport-ebpf.md)

### For Security Reviewers
1. [overview.md](overview.md) §6 Security
2. [security/wireguard-integration.md](security/wireguard-integration.md)
3. [security/sybil-resistance.md](security/sybil-resistance.md)
4. [networking/transport-nat-traversal.md](networking/transport-nat-traversal.md) §11 Abuse Resistance

### For Network Engineers
1. [networking/transport-nat-traversal.md](networking/transport-nat-traversal.md)
2. [networking/io-design.md](networking/io-design.md)
3. [networking/wire-protocol.md](networking/wire-protocol.md)
4. [security/wireguard-integration.md](security/wireguard-integration.md)

---

## Key Concepts

### XOR Distance Metric
The foundation of Kademlia routing. `distance(a, b) = a ⊕ b` (bitwise XOR).

### Shared-Nothing Sharding
State is partitioned across per-core shards using `shard_id = msb_k(selfId XOR key)` to preserve XOR locality without shared locks.

### Zero-Allocation Hot Paths
Network RX/TX, routing table operations, and lookup iterations perform zero heap allocations in steady state via buffer pools and fixed-capacity containers.

### WireGuard Transport Security
All traffic encrypted and authenticated via userspace WireGuard (BoringTun). Silent before handshake; cookie-gated under load.

### Sybil Resistance
Multi-layered defense: proof-of-work on WireGuard ephemeral keys, admission tokens, per-peer quotas, behavioral scoring.

---

## Contributing to Documentation

When adding or updating documentation:
1. Place files in the appropriate subdirectory
2. Update relevant README.md files for navigation
3. Use consistent formatting (see existing docs)
4. Include code examples where helpful
5. Add cross-references to related documents

---

[← Back to Project Root](../)

