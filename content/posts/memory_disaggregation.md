+++
title = "Memory Disaggregation: Decoupling Memory from Compute"
date = 2026-01-05
description = "How modern data centers separate memory pools from compute nodes."
[taxonomies]
tags = ["Memory", "Distributed Systems", "Data Centers", "Hardware"]
+++

---

This paper from VMware Research caught my attention because it asks a question I'd been circling around: why hasn't memory disaggregation happened already?

The idea has been around since the 90s. Intel pushed Rack Scale Architecture in 2013. But it never took off. The authors argue that two things finally align now: a burning economic problem and feasible technology to solve it.

---

## the core idea

Traditional servers bundle everything (CPU, memory, storage) into one box. If you need more RAM, you buy a bigger box or add DIMMs (if you haven't hit the motherboard limit). If you don't use all your RAM, it sits idle.

Memory disaggregation pulls memory out into separate pools that multiple servers can access. Think of it like the shift from local storage to SAN/NAS, but for memory, or better yet, like a GPU rack but for memory.

This gives you two things:

**Capacity expansion.** A server can use more memory than it physically contains by accessing the pool. Similar to Infiniswap, but with hardware support instead of software paging.

**Data sharing.** In the limit, pool memory can be mapped into multiple hosts so they can load/store into the same bytes without explicit send/recv. You still need software protocols (ownership, synchronization, failure), but the access looks like memory instead of messages.

---

## why now

**The economics are painful.** Memory makes up 50% of server cost and 37% of total cost of ownership in cloud environments. Three companies control DRAM production. Demand explodes from data centers, ML training, and in-memory databases. Meanwhile, clusters waste memory (the paper shows over 70% of the time, more than half of aggregate cluster memory sits unused while some machines page to disk).

**The technology finally exists.** RDMA gives you single-digit microsecond latencies. But the bigger enabler is CXL (Compute eXpress Link): cache-coherent load/store access to devices over PCIe, plus a roadmap toward switching, pooling, and (eventually) shared memory fabrics.

Neat detail: the pool can use cheaper, denser (and potentially slower) DRAM, because it's already the "slow tier" compared to local DIMMs.

---

## what's interesting

**It's not just about capacity.** Most remote memory systems (Infiniswap, Fastswap) focus on page-to-remote-RAM-instead-of-disk. Useful, but limited. The promise of CXL is memory that looks like memory: load/store access to a larger pool, and (with the right fabric features) the possibility of mapping the same bytes into multiple hosts. That's qualitatively different from "remote paging."

**The OS problems are hard.** The paper is mostly about what's _unsolved_: memory allocation at scale, scheduling with memory locality, pointer sharing across servers, failure handling for "optional" memory, security for hot-swappable memory pools. These aren't incremental fixes; they require rethinking fundamental abstractions.

**The timeline matches storage disaggregation.** Start small (a few hosts per pool), add switches for rack-scale, and eventually push the fabric boundary outward. Whether that ends up looking like "CXL over X" or something else is still an open question, but the trajectory rhymes with how storage disaggregation played out.

---

## where it works / where it doesn't

**Good fit:**

- Data-intensive workloads (Spark, Ray, distributed DBs) that spend cycles serializing and copying
- Workloads with working sets that barely fit in local memory
- Clusters with significant memory imbalance
- Environments where memory is 50%+ of server cost

**Bad fit:**

- Workloads that fit comfortably in local memory (you'd be adding latency for no benefit)
- Latency-sensitive applications that can't tolerate hundreds of extra nanoseconds in their hot path
- Traditional applications that don't share data across processes

The performance trade-off: pool memory is slower than local (hundreds of ns versus ~100ns), but still orders of magnitude faster than SSD/HDD. For workloads that currently page to disk, this can be transformative. For workloads that don't, adding a slower tier may just hurt.

---

## notes

- Paper: [Aguilera et al., "Memory disaggregation: why now and what are the challenges", ACM SIGOPS Operating Systems Review, 2023](https://dl.acm.org/doi/10.1145/3606557.3606563)
- This is a position paper (no benchmarks, but clear analysis of the problem space)
- CXL 1.0: local memory expansion cards (shipping now)
- CXL 2.0/3.0: fabric switches for pool memory (3-5 years out)
- Latency estimates: local ~100ns, CXL local ~200-300ns, CXL pool ~500-1000ns, RDMA ~1-5μs, SSD ~100μs
- Memory population rules (balanced channels, identical DIMMs) make incremental upgrades nearly impossible (another driver for disaggregation)
- Distributed shared memory systems from the 90s taught us: cache coherence doesn't scale beyond rack-scale
- Security concern: DRAM retains data residue after power-down, and pool memory is hot-swappable (encryption matters more than for local memory)
- Related systems: Infiniswap (software paging over RDMA), LegoOS (full hardware disaggregation), The Machine (HPE, discontinued)
