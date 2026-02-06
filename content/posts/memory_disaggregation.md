+++
title = "Memory Disaggregation"
date = 2026-01-05
description = "What memory disaggregation is and why people are talking about it now."
[taxonomies]
tags = ["Memory", "Distributed Systems"]
+++

I've been reading about memory disaggregation lately and wanted to write down what I understand so far. This is mostly from a VMware Research paper that asks a question I found interesting: why hasn't memory disaggregation happened already?

The idea has been around since the 90s. Intel pushed Rack Scale Architecture in 2013. It didn't take off. The paper argues two things finally align now: the economics are painful enough and the technology exists to actually do something about it.

## the basic idea

Traditional servers bundle CPU, memory, storage into one box. Need more RAM? Buy a bigger box or add DIMMs if you haven't hit the motherboard limit. Don't use all your RAM? It sits idle. You can't share it.

Memory disaggregation pulls memory out into separate pools that multiple servers can access. Think about how we went from local storage to SAN/NAS, but for memory instead.

So what does this actually give you:

- **Capacity expansion.** A server can use more memory than it physically contains by reaching into the pool. Similar to what Infiniswap does, but with hardware support instead of software paging tricks.

- **Data sharing.** Pool memory can be mapped into multiple hosts at once. They can load/store to the same bytes without serializing everything into messages. You still need software to handle ownership and synchronization and failures. But the access itself looks like memory, not network messages.

## why is this coming up now

The economics are getting painful. Memory is like 50% of server cost and 37% of TCO. Three companies control DRAM production. Demand is exploding from data centers and ML and in-memory databases. And here's the frustrating part: clusters waste a lot of memory. Over 70% of the time, more than half of aggregate memory sits unused while some machines are paging to disk because they ran out.

The technology also finally exists. RDMA gives single-digit microsecond latencies. But the bigger thing is CXL which gives you cache-coherent load/store access over PCIe. Plus theres a roadmap toward switches and pooling and shared memory fabrics.

The pool can even use cheaper denser slower DRAM since it's already the "slow tier" compared to local DIMMs anyway.

## what I found interesting

It's not just about capacity. Most remote memory systems like Infiniswap focus on paging to remote RAM. That's useful but limited. CXL promises memory that actually looks like memory: load/store access to a larger pool. With the right fabric features you could map the same bytes into multiple hosts. That's different from shipping pages around.

The OS problems are hard though. The paper is mostly about what's unsolved. Memory allocation at scale. Scheduling with memory locality. Pointer sharing across servers. Failure handling for "optional" memory. Security for hot-swappable pools. These need fundamental rethinking.

The timeline matches what happened with storage disaggregation. Start small with few hosts per pool. Add switches for rack-scale. Push the fabric boundary outward. Whether it ends up being "CXL over something" or something else is open. But the trajectory rhymes with how storage disaggregation went.

## where it fits and where it doesn't

Good for: data-intensive workloads like Spark or Ray or distributed DBs that spend cycles serializing and copying. Working sets that barely fit in local memory. Clusters with memory imbalance. Places where memory is already 50%+ of server cost.

Bad for: workloads that already fit in local memory (you're just adding latency for no reason). Latency-sensitive apps that can't handle hundreds of extra nanoseconds. Traditional apps that don't share data across processes anyway.

Pool memory is slower than local (hundreds of ns vs ~100ns). But still way faster than SSD or disk. For workloads that currently page to disk, this could be big. For workloads that dont page at all, adding a slower tier might just make things worse.

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
