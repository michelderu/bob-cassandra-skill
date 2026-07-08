# Cassandra Architecture Reference

## Deployment options

| Dimension | Apache Cassandra (OSS) | DataStax HCD | DataStax Astra DB |
|---|---|---|---|
| **Operations** | Your team | Your team (vendor-supported stack) | Vendor-managed service |
| **Where it runs** | Your choice (bare metal, VMs, K8s) | On-prem / your cloud (Kubernetes-native) | DataStax-managed public cloud |
| **Best when** | Full DIY control, no vendor lock-in | Self-managed + enterprise support, regulated/hybrid environments | Fastest time-to-value, minimal ops headcount |

**When Cassandra is a good fit:** scale-out with predictable low latency, high availability, global distribution, high-velocity ingest.

**When it may be a poor fit:** rich ad-hoc joins across arbitrary tables; default serializable ACID across many keys (Cassandra 6 introduces Accord for full ACID); tiny datasets with no HA requirement.

---

## Masterless topology & consistent hashing

- Every node is a **peer** — no primary or metadata master. Any node can coordinate reads and writes.
- The **partition key** is hashed to a **token** on the ring; **vnodes** (default 16 per node) spread each physical node across many small token ranges so rebalancing is smooth when nodes join or leave.
- **NetworkTopologyStrategy** (NTS) sets per-DC replica counts for real multi-DC deployments; `SimpleStrategy` is for single-DC only.
- Use `GossipingPropertyFileSnitch` in production so each node knows its rack and DC for replica placement across failure domains.
- `nodetool getendpoints <ks> <table> <partition_key>` shows which nodes hold replicas for a given partition.

---

## Gossip & failure detection

- Gossip is an epidemic protocol: each round a node exchanges **membership, health, load, and schema/token-map** metadata with a few peers; updates spread quickly across large clusters.
- Failure detection uses **phi-accrual** (a suspicion score, not a hard timeout) to avoid false positives under transient network jitter.
- Gossip is **control-plane** metadata — not the application data path; it supports routing and repair decisions.

---

## CAP theorem & tunable consistency

Cassandra is **AP-leaning**: it prioritises availability and partition tolerance; replicas **eventually** converge via replication, repair, and tunable consistency.

**R + W > RF** guarantees overlapping quorums — a reader is certain to see the latest write:

| Strategy | Formula | Result |
|---|---|---|
| Eventual | R + W ≤ RF | Read and write sets may never overlap — stale reads possible |
| Strong | R + W > RF | Quorums overlap — latest write is always visible |

### Consistency levels

| Level | Behaviour |
|---|---|
| **ONE** | One replica acknowledgement — lowest latency, higher staleness risk |
| **QUORUM** | Majority of replicas (e.g. 2 of 3 when RF=3) — balanced latency/correctness |
| **LOCAL_QUORUM** | Quorum within the coordinator's DC only — avoids cross-region latency |
| **ALL** | Every replica — strongest consistency; any down node fails the operation |

`LOCAL_QUORUM` is the recommended default for single-DC and most multi-DC production deployments.

---

## Lightweight Transactions (LWT) & Cassandra 6 Accord

### LWT
- Provides **linearizable** conditional updates (`IF`, `IF NOT EXISTS`) via Paxos-style rounds.
- **Cost**: ≈4× round-trips vs a normal write — use only where strictly needed.
- **Trap**: hot partition + frequent LWT → Paxos contention and tail latency. Prefer idempotent design where possible.

### Cassandra 6 — Accord (ACID transactions)
- Cassandra 6 introduces the **Accord** protocol: full ACID distributed transactions — atomic, consistent, isolated, durable — directly in Cassandra.
- Accord supports multi-row and multi-partition transactions with strong guarantees, removing many workarounds previously required for transactional workloads.
- For Cassandra 6+ workloads, prefer Accord over LWT for transactional requirements.
