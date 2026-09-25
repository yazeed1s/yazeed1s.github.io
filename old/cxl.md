+++
title = "CXL: Why Datacenter Memory is Getting a New Tier"
date = 2026-01-27
description = "CXL creates a memory tier between local DRAM and everything else."
[taxonomies]
tags = ["memory", "CXL", "hardware"]
+++

DRAM can be half of server cost, and a lot of it still sits idle. One machine thrashes while the one next to it uses 30% of its memory. CXL is trying to fix that mismatch.

## the problem

Memory in datacenters is expensive and wasted at the same time. DRAM can be 50% of server cost, and research keeps showing that utilization is terrible. One machine is thrashing because it ran out of memory while another machine next to it is sitting at 30% usage. Some papers claim 70% of aggregate memory is underutilized across a cluster.

The obvious thought is to share memory across machines the way we do with storage, pool it, so if one server has idle memory another server could use it. The issue is that storage can tolerate milliseconds of latency and memory can't, since a cache miss to local DRAM is around 100ns and going over the network with RDMA puts you at 1-5 microseconds, which is 10-50x slower, and for memory access patterns that's a lot. CXL is supposed to bridge this gap.

## what it actually is

CXL stands for Compute Express Link. It runs on the PCIe physical layer, same cables and slots, but with a different protocol on top.

What matters is that it's cache-coherent. The CPU can do normal load/store to CXL-attached memory through the same memory path, without special APIs, RDMA-style verbs, or memory registration. The memory controller treats it like another memory region, basically another NUMA node.

There are three protocols in the spec. CXL.io is basically just PCIe, for device discovery and config, boring stuff. CXL.cache lets devices cache host memory, useful for accelerators. CXL.mem is the interesting one, it lets the host access device-attached memory with load/store.

CXL 1.0 and 1.1 are mostly local expansion. You plug a CXL card with DRAM into a PCIe slot and your system sees more memory. Latency is higher than native DIMMs, maybe 200-300ns instead of 100ns, but it's still memory, not storage. CXL 2.0 adds switching so multiple hosts can share a memory pool. CXL 3.0 goes further with fabric and shared memory semantics.

## mixing memory types

Normally your CPU's memory controller dictates what DRAM you can use. If it's a DDR5 system, all your DIMMs have to be DDR5. Same speed, same density rules, same timing specs. You can't just plug DDR4 into a DDR5 slot.

CXL breaks this because the CXL device has its own memory controller, so it can use whatever DRAM it wants, DDR4, DDR5, older cheaper stuff, slower but denser modules, and the CPU doesn't care since it just sees CXL memory at some address range. So you could keep local DDR5 for hot data and use a CXL card with cheaper DDR4 as a slower tier, or use high-capacity modules that wouldn't fit your motherboard timing rules, which is nice cost-wise because you're not locked to whatever generation the motherboard supports. The tradeoff is latency since CXL adds overhead, but if you're using it for capacity expansion rather than latency-critical paths maybe that's acceptable.

## why not just use RDMA

RDMA is a different model where you need explicit verbs, you post work requests and poll completions, it's not transparent load/store, and you have to register memory, pin pages, and exchange keys, and one-sided operations are async so you don't know when remote writes land unless you add signaling, and latency is around 1-5μs which is fast for networking but slow for memory access patterns. CXL at 200-500ns for pooled memory is closer to local DRAM territory and it's transparent to software, so your malloc can return CXL memory and the application doesn't know the difference. That's the promise anyway, the hardware shipping today is mostly local expansion cards rather than pooled memory, and the pooling stuff is still coming, hopefully.

## the latency thing

Local DRAM is ~100ns, CXL local expansion is ~200-300ns, and CXL through a switch to a shared pool is ~500-1000ns, so pooled CXL is 5-10x slower than local, which isn't nothing, and for tight loops constantly hitting memory that seems expensive. The pitch is that it's still way better than swapping to SSD at 100μs and you get more capacity, which is true. The mental model is tiering, so hot data lives in local DRAM, warm data in the CXL pool, cold data on SSD, and the kernel or runtime migrates pages based on access patterns.

Linux already has machinery for this. NUMA balancing, DAMON for access pattern detection, tiered memory support that got merged recently. Whether this works well in practice with real workloads, I don't know yet. The theory sounds reasonable but there will be edge cases.

## shared memory across hosts

CXL 3.0 talks about multiple hosts accessing the same memory with hardware-maintained cache coherence.

This sounds amazing and also scary at the same time. Cache coherence doesn't scale, the distributed shared memory people learned that in the 90s, and beyond a few nodes the coherence traffic overwhelms everything. The CXL spec people know this, so the scope is limited, maybe a rack, maybe a pod, maybe less, and the vision isn't coherent memory across the whole datacenter, it's more that within a small group of machines you can have shared memory semantics and beyond that you're back to message passing or RDMA. Even rack-scale shared memory is interesting though, since you could have databases sharing buffer caches across replicas or ML training jobs sharing model weights, so there are use cases.

## what's actually shipping

CXL 1.1 memory expanders exist today from Samsung, SK Hynix and others, and Intel Sapphire Rapids supports CXL, and these are mostly used to add capacity to memory-hungry workloads. CXL switches aren't really production-ready yet, there are some prototypes, and I'd guess pooled CXL deployments are 2-3 years out. So when papers say "CXL will enable this" they're often talking about future hardware, and the concepts are solid but the ecosystem is young, so it's worth understanding now because it's coming, just don't expect to deploy pooled CXL next month.

## what I'm still uncertain about

Latency tradeoffs are the big one, since 5-10x slower than local is real overhead, and it's better than SSD but memory-intensive applications might just thrash the CXL tier and make things worse, so the tiering policies need to actually work. Ecosystem maturity is another, because RDMA took years to get right and CXL is newer, so drivers, kernel support, allocation policies, and debugging tools all need to catch up. And it's not clear who benefits, since big cloud providers with massive memory imbalance probably see value while smaller deployments might not see ROI at current hardware costs.

## notes

- CXL consortium includes Intel, AMD, ARM, NVIDIA, Samsung and others
- Built on PCIe 5.0/6.0 physical layer, same slots and cables
- Latency numbers vary by source and topology
- Linux kernel has CXL support in drivers/cxl/, device enumeration works, memory tiering is evolving
- Related specs: Gen-Z (seems dead), CCIX (absorbed into CXL)
- Good starting point: [CXL Consortium](https://www.computeexpresslink.org/)
- For context on memory disaggregation: Aguilera et al., "Memory disaggregation: why now and what are the challenges"
