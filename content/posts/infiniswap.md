+++
title = "Infiniswap"
date = 2026-01-18
description = "How Infiniswap uses RDMA to swap pages to remote memory instead of disk."
[taxonomies]
tags = ["memory", "RDMA", "paging"]
+++

I found this paper while reading about memory disaggregation. The idea is simple: when a machine runs out of RAM, page to another machine's unused memory instead of disk.

What caught my attention is how they did it: it works without application changes or core kernel patches through a kernel module that hooks into Linux's swap path, where remote RAM becomes the fast tier and disk is just the fallback.

## the problem they're solving

Production clusters waste a lot of memory. Some machines are memory-starved while others sit idle. The 99th percentile machine uses 2-3× more memory than the median. Over half the cluster's aggregate memory goes unused.

When apps can't fit their working set in RAM, performance falls off a cliff. VoltDB drops from 95K TPS to 4K TPS. Memcached's tail latency shoots up 21×. Disk is just too slow (like 1000× slower than memory).

So they thought: RDMA gives single-digit microsecond latencies. That's fast enough to make remote memory a viable swap target. Pages go to remote RAM over RDMA instead of disk. The remote CPU stays out of the data movement entirely since the RNIC does the DMA.

Result: swap that looks normal to Linux but is backed by slabs of remote memory scattered across the cluster.

## what I thought was clever

**Using swap as the integration point.** Instead of modifying the page fault handler or remapping virtual memory, they plug into Linux's swap subsystem. The kernel already knows how to page out and page in. Infiniswap just changes where those pages live. The trade-off is you still go through the swap path (page faults, context switches). But you get deployment simplicity because everything else just works.

**One-sided RDMA.** Traditional network block devices like Mellanox's nbdX use send/recv. Remote CPU wakes up, copies data, responds. Infiniswap uses RDMA_READ and RDMA_WRITE. The RNIC accesses remote memory directly without running any code on the remote side. nbdX burns multiple vCPUs on the remote machine. Infiniswap doesn't touch the remote CPU at all.

**Slab-based design.** Pages are grouped into 1GB slabs. Each slab maps to one remote machine. This keeps metadata manageable. Tracking millions of 4KB pages across the cluster would be expensive. Hot slabs (more than 20 page I/O ops/sec) get mapped to remote memory. Cold slabs stay on disk.

## where it works well

Memory-bound workloads see big wins. Memcached stays nearly flat even when only 50% of the working set fits in memory. PowerGraph runs 6.5× faster. VoltDB sees 15× throughput improvement over disk.

Cluster memory utilization: goes from 40% to 60%. That's 47% more effective use of RAM. Network overhead is less than 1% of capacity.

## where it doesn't work

CPU-bound workloads don't benefit much. VoltDB and Spark already run at high CPU utilization. Adding paging overhead (context switches, TLB flushes, page table walks) eats into that. Spark at 50% memory thrashes so badly it doesn't complete.

There's a fundamental limit here: this isn't local memory. Page faults still happen. You're masking latency, not eliminating it. For workloads where microseconds matter deterministically, that's still a problem.

## notes

- Paper: [Gu et al., "Efficient Memory Disaggregation with Infiniswap", NSDI 2017](https://www.usenix.org/system/files/conference/nsdi17/nsdi17-gu.pdf)
- Tested on 32 machines, 56 Gbps Infiniband, 64GB RAM each
- Slab placement uses "power of two choices" (pick two random machines, query free memory, use the one with more headroom)
- Slab eviction queries E+5 machines, evicts coldest (~363μs median)
- Page-out: synchronous RDMA_WRITE + async disk write (disk is fallback if remote crashes)
- Page-in: check bitmap → RDMA_READ if remote, else disk
- Slab remapping after failure takes ~54ms (Infiniband memory registration)
- Default headroom threshold: 8GB per machine
- Hot slab threshold: 20 page I/O ops/sec (EWMA, α=0.2)
- Compared against nbdX (Mellanox), Fastswap, LegoOS
- Code: [SymbioticLab/infiniswap](https://github.com/SymbioticLab/Infiniswap)
