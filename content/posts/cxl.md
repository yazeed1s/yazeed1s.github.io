+++
title = "CXL: Compute Express Link"
date = 2026-01-12
description = "CXL turns PCIe into a coherent memory link, enabling memory expansion and (eventually) pooling."
[taxonomies]
tags = ["CXL", "Memory", "Hardware", "PCIe", "Data Centers"]
+++

---

CXL is the thing I keep seeing in memory disaggregation discussions that _isn't_ RDMA.

The core idea is simple:

**CXL lets a CPU talk to devices over PCIe as if some of that device memory is just "more memory" (with cache coherence), not just DMA behind a driver.**

That sounds like magic until you remember the trade-off: it's still farther than local DRAM. You're buying capacity (and sometimes sharing/pooling) by paying extra latency and giving the OS a harder placement problem.

---

## what cxl actually adds

PCIe already lets devices do DMA. That's not the interesting part.

The interesting part is **coherence + memory semantics**:

- With plain PCIe, the host talks to devices via MMIO, and devices talk to host memory via DMA. You can move bytes around, but it doesn't look like "shared memory" (and devices don't get to participate as coherent caching agents).
- With CXL, the link can carry transactions that participate in the host's coherency domain (depending on mode/device), so "who sees which bytes when" becomes something hardware can help enforce.

This is why CXL is pitched as a _memory_ interconnect, not just an I/O bus.

---

## the three protocols (the only ones worth remembering)

CXL isn't one protocol, it's three riding on top of PCIe:

- **CXL.io**: the boring compatibility layer (enumeration, config space, interrupts). Basically "PCIe-like."
- **CXL.cache**: lets a device cache host memory coherently (device acts like a coherent agent).
- **CXL.mem**: lets the host access device-attached memory with load/store semantics (device memory looks like memory, not an I/O buffer).

If you only care about memory expansion, **CXL.mem** is the star.

---

## device types (why "type 3" shows up everywhere)

The spec groups devices into types. You don't need the full taxonomy, just the mental model:

- **Type 1/2**: accelerators that want coherency (think "devices that want to touch host memory without fighting the cache hierarchy").
- **Type 3**: **memory expanders**. This is the disaggregation-adjacent one: plug more capacity behind the link and make it usable by the host.

Type 3 is where the "add memory without adding sockets" story comes from.

---

## what this looks like to software

If CXL is doing its job, _applications don't change_, but the OS has new headaches.

In a typical "memory expansion" setup, CXL memory shows up as:

- a new pool of capacity the kernel can allocate from
- often with NUMA-like properties (different latency/bandwidth than local DRAM)
- sometimes managed as a lower tier (hot things stay local, colder things spill)

So the system-level question becomes:

> "Which bytes should live in local DRAM, and which bytes can tolerate being slower?"

If you let the OS guess, you get "it works, but sometimes it hurts." If you want predictable performance, you usually need policies: NUMA placement, tiering, explicit pinning, or application-level caching.

---

## cxl vs rdma (why they feel similar and why they're not)

Both get used to talk about "remote memory." The similarity ends there.

- **RDMA**: you explicitly issue operations over the network (READ/WRITE/SEND). It's fast (microseconds), but it's still a network, and you still build protocols on top.
- **CXL**: the CPU can issue load/stores into device memory, and hardware can help keep things coherent. It's closer to "NUMA, but farther."

Also: CXL is a PCIe link/fabric. It's built for short-reach inside a server (and maybe rack-scale switching), not "put it on Ethernet and forget about it."

RDMA is a way to move bytes fast. CXL is a way to make _some_ bytes look like memory.

---

## where cxl wins

- **Capacity expansion without a bigger server.** You want more memory, but you don't want another socket just to get more DIMM slots.
- **Tiered memory.** You have hot working sets and cold-but-still-useful data. CXL can be the "bigger, slower tier" that beats going to SSD.
- **Disaggregation building block.** If you ever want pooled memory, you need something like CXL on the inside before you can make the outside story real.

## where it hurts

- **Latency-sensitive hot paths.** If your workload is dominated by pointer-chasing and cache misses, adding a slower tier can destroy tail latency.
- **Bandwidth and congestion.** It's easy to sell "more capacity." It's harder to deliver "more bandwidth" when many cores start leaning on the same link/switch.
- **Software maturity and policy.** You can make it work without app changes, but getting good performance is a placement problem, and placement is where systems go to die.

---

## notes

- If you remember one sentence: CXL is "PCIe, but with coherence and memory semantics."
- CXL doesn't eliminate NUMA; it gives you a new kind of NUMA-shaped problem.
- The exciting part (pooling/sharing across a fabric) is also the part with the most sharp edges: security, isolation, failure handling, and who gets to be coherent with whom.
