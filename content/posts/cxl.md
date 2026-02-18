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

The obvious thought: why not share memory across machines, like we do with storage. Pool it. If one server has idle memory another server could use it.

But the issue is that storage can tolerate milliseconds of latency. Memory can't. A cache miss to local DRAM is around 100ns. Go over the network with RDMA and you're at 1-5 microseconds. That's 10-50x slower. For memory access patterns that's a lot.

CXL is supposed to bridge this gap.

## what it actually is

CXL stands for Compute Express Link. It runs on the PCIe physical layer, same cables and slots, but with a different protocol on top.

What matters is that it's cache-coherent. The CPU can do normal load/store to CXL-attached memory through the same memory path, without special APIs, RDMA-style verbs, or memory registration. The memory controller treats it like another memory region, basically another NUMA node.

There are three protocols in the spec. CXL.io is basically just PCIe, for device discovery and config, boring stuff. CXL.cache lets devices cache host memory, useful for accelerators. CXL.mem is the interesting one, it lets the host access device-attached memory with load/store.

CXL 1.0 and 1.1 are mostly local expansion. You plug a CXL card with DRAM into a PCIe slot and your system sees more memory. Latency is higher than native DIMMs, maybe 200-300ns instead of 100ns, but it's still memory, not storage. CXL 2.0 adds switching so multiple hosts can share a memory pool. CXL 3.0 goes further with fabric and shared memory semantics.

## mixing memory types

Normally your CPU's memory controller dictates what DRAM you can use. If it's a DDR5 system, all your DIMMs have to be DDR5. Same speed, same density rules, same timing specs. You can't just plug DDR4 into a DDR5 slot.

CXL breaks this because the CXL device has its own memory controller. It can use whatever DRAM it wants. DDR4, DDR5, older cheaper stuff, slower but denser modules. The CPU doesn't care. It just sees CXL memory at some address range.

So you could keep local DDR5 for hot data and use a CXL card with cheaper DDR4 as a slower tier, or use high-capacity modules that wouldn't fit your motherboard timing rules. Cost-wise this is nice because you're not locked to whatever generation the motherboard supports.

The tradeoff is latency. CXL adds overhead. But if you're using it for capacity expansion rather than latency-critical paths, maybe that's acceptable.

## why not just use RDMA

RDMA is a different model. You need explicit verbs, post work requests, poll completions. It's not transparent load/store. You have to register memory, pin pages, exchange keys. One-sided operations are async so you don't know when remote writes land unless you add signaling. And latency is around 1-5μs which is fast for networking but slow for memory access patterns.

CXL at 200-500ns for pooled memory is closer to local DRAM territory. And it's transparent to software. Your malloc can return CXL memory and the application doesn't know the difference.

That's the promise anyway. The hardware shipping today is mostly local expansion cards, not pooled memory. The pooling stuff is still coming hopefully.

## the latency thing

Local DRAM is ~100ns. CXL local expansion is ~200-300ns. CXL through a switch to a shared pool is ~500-1000ns.

So pooled CXL is 5-10x slower than local. That's not nothing. For tight loops constantly hitting memory, that seems expensive. The pitch is that it's still way better than swapping to SSD (100μs) and you get more capacity. Which is true.

The mental model is tiering: hot data in local DRAM, warm data in the CXL pool, cold data on SSD, and the kernel (or runtime) migrating pages based on access patterns.

Linux already has machinery for this. NUMA balancing, DAMON for access pattern detection, tiered memory support that got merged recently. Whether this works well in practice with real workloads, I don't know yet. The theory sounds reasonable but there will be edge cases.

## shared memory across hosts

CXL 3.0 talks about multiple hosts accessing the same memory with hardware-maintained cache coherence.

This sounds amazing and also scary at the same time.

Cache coherence doesn't scale. The distributed shared memory people learned this in the 90s. Beyond a few nodes the coherence traffic overwhelms everything.

The CXL spec people know this. The scope is limited, maybe a rack, maybe a pod, maybe less. The vision isn't coherent memory across the whole datacenter. It's more like, within a small group of machines you can have shared memory semantics. Beyond that you're back to message passing or RDMA.

Even rack-scale shared memory is interesting though. Databases that want to share buffer caches across replicas. ML training jobs that share model weights. There are use cases.

## what's actually shipping

CXL 1.1 memory expanders exist today from Samsung, SK Hynix and others. Intel Sapphire Rapids supports CXL. These are mostly used to add capacity to memory-hungry workloads.

CXL switches are not really production-ready yet. Some prototypes. I'd guess pooled CXL deployments are 2-3 years out.

So when papers say "CXL will enable this," they're often talking about future hardware. The concepts are solid but the ecosystem is young. Worth understanding now because it's coming, but don't expect to deploy pooled CXL next month.

## what I'm still uncertain about

**Latency tradeoffs.** 5-10x slower than local is real overhead. Better than SSD, yes. But memory-intensive applications might just thrash the CXL tier and make things worse. Tiering policies need to actually work.

**Ecosystem maturity.** RDMA took years to get right. CXL is newer. Drivers, kernel support, allocation policies, debugging tools, all of this needs to catch up.

**Who benefits.** Big cloud providers with massive memory imbalance probably see value. Smaller deployments might not see ROI at current hardware costs.

## notes

- CXL consortium includes Intel, AMD, ARM, NVIDIA, Samsung and others
- Built on PCIe 5.0/6.0 physical layer, same slots and cables
- Latency numbers vary by source and topology
- Linux kernel has CXL support in drivers/cxl/, device enumeration works, memory tiering is evolving
- Related specs: Gen-Z (seems dead), CCIX (absorbed into CXL)
- Good starting point: [CXL Consortium](https://www.computeexpresslink.org/)
- For context on memory disaggregation: Aguilera et al., "Memory disaggregation: why now and what are the challenges"
