+++
title = "CXL: Why Datacenter Memory is Getting a New Tier"
date = 2026-02-05
description = "CXL creates a memory tier between local DRAM and everything else."
[taxonomies]
tags = ["Memory", "CXL", "Hardware"]
+++

---

CXL keeps appearing everywhere I read about memory disaggregation. Every paper mentions it as _"the future"_ of remote memory access. And I think I finally sat down to understand what it actually is.

---

## the problem it's solving

Memory is expensive, especially in datacenters. DRAM can be 50% of server cost. And it's wasted constantly. One machine thrashes while another sits at 30% usage. Many research papers have shown that memory usage is highly imbalanced across servers in a datacenter, and memory underutilization is 70% in some cases. So what do we do? 

The obvious fix is to let machines share memory. Pool it like storage. So instead of the memory setting idle in one server, it can be used by another server. 

But memory access is different from storage. Storage can tolerate milliseconds. Memory access happens in nanoseconds. A cache miss to DRAM is ~100ns. Going over the network adds orders of magnitude. RDMA gets you down to single-digit microseconds. Still 10-50x slower than local memory.

CXL is supposed to close that gap.

---

## what cxl actually is

CXL stands for Compute Express Link. It's a standard built on top of PCIe physical layer. You use the same cables and slots.

The key thing: it's cache-coherent. The CPU can do normal load/store to CXL-attached memory. The memory controller handles it. No special APIs. No registration like RDMA. Just... memory.

Three protocols in CXL:

**CXL.io** — basically PCIe. For discovery, configuration, interrupts. Boring stuff.

**CXL.cache** — lets devices cache host memory. Useful for accelerators that want to share data with the CPU without explicit copies.

**CXL.mem** — the interesting one. Lets the host access device-attached memory. The CPU sees it as another memory region. Different NUMA node, slower latency, but addressable like regular RAM.

CXL 1.0/1.1 is mostly expansion cards. Plug a CXL card with extra DRAM into PCIe slot. Your system gets more memory. Latency is worse than local DIMMs (maybe 200-300ns instead of 100ns), but it's still memory, not storage.

CXL 2.0 adds switching. Multiple hosts can connect to a shared memory pool. CXL 3.0 pushes this further with fabric.

---

## mixing memory types

This is the part that took me a while to get.

Normally, your CPU's memory controller dictates what DRAM you use. DDR5 system? All your DIMMs are DDR5. Same speed, same density rules. You can't just plug DDR4 into a DDR5 slot.

CXL breaks this. The CXL device has its own memory controller. It can use whatever DRAM it wants. DDR4, DDR5, older stuff, slower but denser chips. The CPU doesn't care. It just sees CXL memory at some address range.

So you could have a system with local DDR5 for hot data, and a CXL card with cheaper DDR4 as a slower tier. Or high-capacity, high-density modules that wouldn't work in your native DIMM slots.

This is interesting from a cost angle. You're not stuck with whatever generation your motherboard supports. Want to keep using old DRAM you already bought? Stick it behind CXL. Want the newest high-density stuff that doesn't fit your memory controller's timing specs? CXL device can handle it.

The tradeoff is latency. CXL adds overhead. But if you're using it for capacity expansion (not latency-critical paths), maybe that's fine.

---

## why not just use rdma

I kept wondering this. RDMA already gives you fast remote memory access. Infiniswap uses it. Works today.

But RDMA is different:

1. You need explicit verbs. Post work request, poll completion. Not load/store.
2. Memory registration overhead. Pin pages, get keys, share keys.
3. One-sided ops are async. CPU doesn't know when remote writes land unless you signal.
4. ~1-5μs latency. Fast for network, slow for memory.

CXL is supposed to be ~200-500ns for pooled memory. Still slower than local, but closer. And it's transparent to software. Your malloc can return CXL memory. The app doesn't know.

This is the promise anyway. Shipping hardware today is mostly local expansion, not pooled.

---

## the latency question

This is where I'm not fully convinced.

Local DRAM: ~100ns. CXL local expansion: ~200-300ns. CXL pooled (through switch): ~500-1000ns.

Okay so CXL pool is 5-10x slower than local. That's... a lot? For some workloads, maybe fine. For tight loops hitting memory constantly, this seems expensive.

The pitch is: better than swapping to SSD (100μs) or going over RDMA. And you get more capacity.

I think the mental model is tiering. Hot data in local DRAM. Warm data in CXL pool. Cold data in SSD or remote. The kernel or some runtime migrates pages between tiers based on access patterns.

Linux already has this machinery. NUMA balancing, DAMON, tiered memory support merged recently. Whether it works well in practice is another question.

---

## cache coherence across hosts

CXL 3.0 talks about shared memory. Multiple hosts access same bytes. Hardware maintains coherence.

This sounds amazing and also scary.

Cache coherence doesn't scale. We learned this from distributed shared memory systems in the 90s. Beyond a few nodes, the coherence traffic kills you.

CXL spec people know this. The scope is limited. Maybe a rack. Maybe less. The vision isn't "coherent memory across datacenter." It's more like: within a pod or blade, you get shared memory semantics. Beyond that, you're back to message passing or RDMA-style access.

Still, even rack-scale shared memory is interesting for some workloads. Databases that want to share buffer caches. ML jobs that share model weights.

---

## what's shipping

CXL 1.1 devices exist. Samsung, SK Hynix, others have memory expanders. Mostly used to add capacity to memory-hungry workloads. Intel Sapphire Rapids supports CXL.

CXL switches are... not really there yet. Some prototypes. Production deployments of pooled CXL memory are probably 2-3 years out.

So when papers say "CXL will enable this", they're often talking about future hardware. The concepts matter but the ecosystem is young.

---

## what i'm still unsure about

**Is the latency worth it?** 5-10x slower than local. Yes better than SSD. But memory-intensive apps might just thrash the CXL tier. Need tiering policies that actually work.

**Will the ecosystem mature?** RDMA took years to get right. CXL is newer. Drivers, kernel support, allocation policies, debugging tools. All need to catch up.

**Who actually benefits?** Big cloud providers with massive memory imbalance probably. Smaller deployments might not see ROI with current hardware costs.

---

## notes

- CXL consortium: Intel, AMD, ARM, NVIDIA, Samsung, etc.
- Built on PCIe 5.0/6.0 physical layer. Same slots and cables.
- Latency estimates vary by source. Local expansion seems around 200-300ns. Pooled depends heavily on topology.
- Linux kernel CXL support: drivers/cxl/. Device enumeration works. Memory tiering still evolving.
- Related specs: Gen-Z (dead?), CCIX (absorbed into CXL)
- Good overview: [CXL Consortium spec](https://www.computeexpresslink.org/)
- For memory disaggregation context: Aguilera et al., "Memory disaggregation: why now and what are the challenges"
