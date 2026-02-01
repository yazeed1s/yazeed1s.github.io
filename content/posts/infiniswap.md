+++
title = "Infiniswap: Remote Memory Paging Over RDMA"
date = 2026-01-18
description = "How Infiniswap uses RDMA to swap pages to remote memory instead of disk."
[taxonomies]
tags = ["Memory", "RDMA", "Paging", "Distributed Systems", "Linux"]
+++

---

I came across this paper while looking into memory disaggregation. The idea is deceptively simple: when a machine runs out of RAM, instead of paging to disk, page to another machine's unused memory over the network.

What caught my attention is _how_ they pulled it off; no application changes, no core kernel patching. It's a kernel module that hooks into Linux's swap path and uses remote RAM as the fast tier, with disk as fallback.

---

## the core idea

Production clusters waste a lot of memory. Some machines are memory-starved while others sit idle. The 99th percentile machine often uses 2-3× more memory than the median. Meanwhile, over half the cluster's aggregate memory goes unused.

When applications can't fit their working set in RAM, performance falls off a cliff. VoltDB drops from 95K TPS to 4K TPS. Memcached's tail latency shoots up 21×. Disk is too slow to help (1000× slower than memory).

Infiniswap's insight: RDMA networks give you single-digit microsecond latencies. That's fast enough to make remote memory a viable swap target. Instead of page -> disk, you do page -> remote RAM over RDMA. The remote CPU stays out of the data movement; the RNIC does the DMA.

The result: swap that looks normal to Linux, but is backed by slabs of remote memory scattered across the cluster.

---

## what's clever

A few design choices stood out:

**Using swap as the integration point.** Instead of modifying the kernel's page fault handler or remapping virtual memory, Infiniswap plugs into Linux's existing swap subsystem. The kernel already knows how to page out and page back in. Infiniswap just changes where those swapped pages live. The trade-off is you're still going through the swap path (page faults, context switches, the whole thing) but you get deployment simplicity.

**One-sided RDMA.** Traditional network block devices (like Mellanox's nbdX) use send/recv semantics. The remote CPU has to wake up, copy data, respond. Infiniswap uses RDMA_READ and RDMA_WRITE (the RNIC directly accesses remote memory without running remote code on the critical path). The paper shows nbdX burns multiple vCPUs on the remote side; Infiniswap largely avoids that.

**Slab-based design.** Pages are grouped into 1GB slabs. Each slab maps to one remote machine. This keeps the metadata manageable (tracking millions of individual 4KB pages across the cluster would be expensive). When a slab gets "hot" (>20 page I/O ops/sec), it gets mapped to remote memory. Cold slabs stay on disk.

---

## where it works

Memory-bound workloads see big wins. Memcached stays nearly flat even when only 50% of the working set fits in memory. PowerGraph runs 6.5× faster. VoltDB, while CPU-heavy, still sees 15× throughput improvement over disk.

The cluster memory utilization goes from 40% to 60% (that's 47% more effective use of RAM, with minimal network overhead (<1% of capacity)).

## where it doesn't

CPU-bound workloads don't benefit as much. VoltDB and Spark already run at high CPU utilization. Adding paging overhead (context switches, TLB flushes, page table walks) eats into that. Spark at 50% memory thrashes so badly it doesn't complete.

The fundamental limit: this isn't local memory. Page faults still happen. You're masking latency, not eliminating it. For workloads where microseconds matter deterministically, that's a problem.

---

## notes

- Paper: [Gu et al., "Efficient Memory Disaggregation with Infiniswap", NSDI 2017](https://www.usenix.org/system/files/conference/nsdi17/nsdi17-gu.pdf)
- Tested on 32 machines, 56 Gbps Infiniband, 64GB RAM each
- Slab placement uses "power of two choices" (pick two random machines, query free memory, map to the one with more headroom)
- Slab eviction queries E+5 machines, evicts the coldest from that set (~363μs median)
- Page-out: synchronous RDMA_WRITE + async disk write (disk is fallback if remote crashes)
- Page-in: check bitmap -> RDMA_READ if remote, else read from disk
- Slab remapping after failure takes ~54ms (Infiniband memory registration)
- Default headroom threshold: 8GB per machine
- Hot slab threshold: 20 page I/O ops/sec (EWMA, α=0.2)
- Compared to: nbdX (Mellanox), Fastswap (kernel modification), LegoOS (full OS redesign)
- Code available on GitHub [SymbioticLab/infiniswap](https://github.com/SymbioticLab/Infiniswap)
