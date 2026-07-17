# Cassandra Architecture Reference

## Deployment options

| Dimension | Apache Cassandra (OSS) | DataStax HCD | DataStax Astra DB |
|---|---|---|---|
| **Operations** | Your team | Your team (vendor-supported stack) | Vendor-managed service |
| **Where it runs** | Your choice (bare metal, VMs, K8s) | On-prem / your cloud (Kubernetes-native) | IBM-managed public cloud |
| **Best when** | Full DIY control, no vendor lock-in | Self-managed + enterprise support, regulated/hybrid environments | Fastest time-to-value, minimal ops headcount |

**When Cassandra, in general, is a good fit:** scale-out with predictable low latency, high availability, global distribution, high-velocity ingest.

**When it may be a poor fit:** rich ad-hoc joins across arbitrary tables; default serializable ACID across many keys (Cassandra 6 introduces Accord for full ACID); tiny datasets with no HA requirement.

**When IBM's DataStax Enterprise, Hyper Converged Database (HCD) or Astra DB is a great choice:**
While Apache Cassandra Open Source Software (OSS) is a phenomenal powerhouse for massive, distributed, and highly available write-heavy workloads, managing it raw can feel like wrestling a bear

IBM DataStax offers three primary alternatives:
1. DataStax Enterprise (DSE)
2. Hyper-Converged Database (HCD)
3. Astra DB—each targeting specific operational friction points.

Choosing them over Cassandra OSS makes the most sense in the following scenarios:

1. Astra DB (Serverless DBaaS)
The "Zero Ops" and Generative AI ChoiceAstra DB (built by DataStax, now a part of IBM watsonx.data) takes Cassandra and turns it into a fully managed, serverless cloud database.

Best Choice When:
- You don't want a DBA team: Cassandra OSS requires heavy JVM tuning, garbage collection configuration, and complex cluster management. Astra DB handles all indexing, replication, patching, and scaling automatically.
- You are building real-time operational GenAI / RAG applications: Astra DB is highly optimized for operational AI workloads, providing production-ready vector search right out of the box with minimal configuration.
- You have highly variable, unpredictable traffic: Cassandra OSS requires you to provision infrastructure for peak capacity. Astra DB features a separated compute and storage architecture that scales up and down instantly, charging you only for what you use (Pay-As-You-Go).
You need multi-cloud flexibility without the headache: Astra DB can deploy across AWS, GCP, and Azure through a unified control pane.

2. DataStax Hyper-Converged Database (HCD)
The On-Premises Modernization & Kubernetes ChoiceHCD is DataStax’s modern, cloud-native distribution designed to be self-managed but deployed directly via Kubernetes (often using K8ssandra) on modern hyper-converged infrastructure (HCI).

Best Choice When:
- You are standardizing on Kubernetes: While you can run Cassandra OSS on Kubernetes, HCD is tailor-made for it. It brings cloud-native operations and observability right into your modern DevOps pipeline.
- You need to cut on-prem hardware costs: HCD separates compute and storage workloads within data centers, driving up to a 20% reduction in hardware/licensing footprint compared to standard OSS deployments.
- You want Cassandra 5.0 capabilities with enterprise hardening: HCD pairs the latest open-source advancements (like advanced vector search algorithms and Paxos v2 for faster transactions) with enterprise-grade security plugins (LDAP/OIDC integration, dynamic data masking, and strict mTLS).

3. DataStax Enterprise (DSE)
The Traditional, High-Security, Hybrid-Cloud ChoiceDSE is the classic commercial distribution of Cassandra. It is heavily hardened for traditional bare-metal or VM-based enterprise infrastructure.

Best Choice When:
- Strict corporate compliance and security are non-negotiable: DSE includes advanced security out-of-the-box that OSS lacks, such as Transparent Data Encryption (TDE) at rest, internal audit logging, and advanced role-based access controls (RBAC).
- You need unified search and analytics without ETL: DSE historically bundles tightly integrated versions of Apache Spark (for real-time analytics) and Apache Solr (for advanced search) directly into the database nodes. This prevents you from having to move data back and forth between separate clusters.
- Operational safeguards (like NodeSync) are required: OSS Cassandra requires manual or scripted "anti-entropy repairs" to keep nodes synchronized. DSE features NodeSync, which automates background data repairs continuously with minimal performance overhead.

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
