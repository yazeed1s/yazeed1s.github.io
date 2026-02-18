+++
title = "Memory Disaggregation"
date = 2026-02-02
description = "What memory disaggregation is and why people are talking about it now."
[taxonomies]
tags = ["Memory", "Distributed Systems"]
+++

Memory is expensive, and in some clusters it's half the server cost while a lot of it sits idle. Over 70% of the time more than half of aggregate cluster memory is unused, yet some machines are paging to disk because they ran out.

Memory disaggregation is the idea that you pull memory out of servers, pool it, and let machines use what they need. A VMware Research paper I read recently asks why this hasn't happened already, and their answer is that the economics were bad but tolerable and the technology didn't exist, but now both are changing.

## the basic idea

Traditional servers bundle CPU, memory, and storage into one box. Need more RAM? Buy a bigger box or add DIMMs if you haven't hit the motherboard limit. Don't use all your RAM? It sits idle and you can't share it.

Memory disaggregation pulls memory out into separate pools that multiple servers can access, similar to how we went from local storage to SAN/NAS but for memory instead.

This gives you two things: capacity expansion where a server can use more memory than it physically contains by reaching into the pool (similar to what Infiniswap does but with hardware support instead of software paging tricks), and data sharing where pool memory can be mapped into multiple hosts at once so they can load/store to the same bytes without serializing everything into messages. You still need software to handle ownership and synchronization and failures, but the access itself looks like memory, not network messages.

## why is this coming up now

The economics are getting painful because memory is like 50% of server cost and 37% of TCO, three companies control DRAM production, and demand is exploding from data centers and ML and in-memory databases. And here's the frustrating part: clusters waste a lot of memory where over 70% of the time more than half of aggregate memory sits unused while some machines are paging to disk because they ran out.

The technology also finally exists since RDMA gives single-digit microsecond latencies, but the bigger thing is CXL which gives you cache-coherent load/store access over PCIe, plus there's a roadmap toward switches and pooling and shared memory fabrics. The pool can even use cheaper denser slower DRAM since it's already the "slow tier" compared to local DIMMs anyway.

## what I found interesting

This isn't only about capacity because most remote-memory systems like Infiniswap focus on paging to remote RAM, which helps but stays limited. CXL is aiming for memory that still behaves like memory with load/store access to a larger pool, and with the right fabric features you can even map the same bytes into multiple hosts, which is very different from shipping pages around.

The OS problems are hard though, and the paper mostly focuses on what's still unsolved: memory allocation at scale, scheduling with memory locality, pointer sharing across servers, failure handling for "optional" memory, and security for hot-swappable pools, all of which need fundamental rethinking.

The timeline matches what happened with storage disaggregation: start small with a few hosts per pool, add switches for rack-scale, then push the fabric boundary outward. Whether it ends up being "CXL over something" or something else is open, but the trajectory rhymes with how storage disaggregation went.

## where it fits and where it doesn't

Good for data-intensive workloads like Spark or Ray or distributed DBs that spend cycles serializing and copying, working sets that barely fit in local memory, clusters with memory imbalance, and places where memory is already 50%+ of server cost.

Bad for workloads that already fit in local memory (you're just adding latency for no reason), latency-sensitive apps that can't handle hundreds of extra nanoseconds, and traditional apps that don't share data across processes anyway.

Pool memory is slower than local (hundreds of ns vs ~100ns) but still way faster than SSD or disk. For workloads that currently page to disk this could be big, but for workloads that don't page at all adding a slower tier might just make things worse.

## notes

- Paper: [Aguilera et al., "Memory disaggregation: why now and what are the challenges", ACM SIGOPS, 2023](https://dl.acm.org/doi/10.1145/3606557.3606563)
- Position paper, no benchmarks, just analysis of the problem space
- CXL 1.0: local memory expansion cards (shipping now)
- CXL 2.0/3.0: fabric switches for pool memory (maybe 3-5 years out)
- Latency estimates: local ~100ns, CXL local ~200-300ns, CXL pool ~500-1000ns, RDMA ~1-5μs, SSD ~100μs
- Memory population rules (balanced channels, identical DIMMs) make upgrades nearly impossible in practice
- Distributed shared memory from 90s taught us: cache coherence doesn't scale beyond rack
- Security: DRAM retains data after power-down and pool memory is hot-swappable so encryption matters
- Related work: Infiniswap, LegoOS, The Machine (HPE, discontinued)
