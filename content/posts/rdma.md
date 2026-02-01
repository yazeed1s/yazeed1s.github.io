+++
title = "RDMA: Bypassing the Kernel for Network I/O"
date = 2025-12-28
description = "How RDMA bypasses the kernel on the data path, letting NICs DMA straight between user buffers."
[taxonomies]
tags = ["RDMA", "Networking", "Kernel Bypass", "Systems Programming"]
+++

---

I've been reading about memory disaggregation and kept running into RDMA as the enabling technology. The pitch is simple: what if your application could read and write memory on a remote machine without dragging the remote CPU into the loop?

The core concept is kernel bypass. In traditional TCP, every packet goes through the kernel's network stack. Data gets copied (or at least touched) as it crosses user/kernel boundaries, checksums happen, packets get queued, interrupts fire. The CPU is in the middle of basically everything.

RDMA flips this. The network card (called an RNIC) reads directly from your application's memory buffer and sends it over the wire. The remote RNIC DMA-writes directly into the destination application's buffer.

No kernel on the data path, no socket buffers, and (for one-sided ops) no remote syscall/interrupt just to move bytes. (You still pay CPU to post work and handle completions, but you stop burning cycles on the kernel stack and copies.)

This is why systems like Infiniswap and modern distributed databases obsess over RDMA. Often single-digit microsecond latencies instead of tens or hundreds.

The difference is clear when you visualize the stack:

![TCP vs RDMA](/images/rdma.png)

```text
                   TCP/IP                                      RDMA

+-------------------+  +-------------------+     +-------------------+  +-------------------+
|      SERVER       |  |      SERVER       |     |      SERVER       |  |      SERVER       |
|                   |  |                   |     |                   |  |                   |
|   +-----------+   |  |   +-----------+   |     |   +-----------+   |  |   +-----------+   |
|   |    App    |   |  |   |    App    |   |     |   |    App    |   |  |   |    App    |   |
|   +-----+-----+   |  |   +-----+-----+   |     |   +-----+-----+   |  |   +-----+-----+   |
|         |         |  |         |         |     |         |         |  |         |         |
|         v         |  |         v         |     |         v         |  |         v         |
|   +-----------+   |  |   +-----------+   |     |   +-----------+   |  |   +-----------+   |
|   |  Buffer   |   |  |   |  Buffer   |   |     |   |  Buffer   |   |  |   |  Buffer   |   |
|   +-----+-----+   |  |   +-----+-----+   |     |   +-----+-----+   |  |   +-----+-----+   |
|         |         |  |         |         |     |         |         |  |         |         |
|         v         |  |         v         |     |         |         |  |         |         |
|   +-----------+   |  |   +-----------+   |     |   (kernel bypass) |  |   (kernel bypass) |
|   |  Sockets  |   |  |   |  Sockets  |   |     |         |         |  |         |         |
|   +-----+-----+   |  |   +-----+-----+   |     |         |         |  |         |         |
|         |         |  |         |         |     |         |         |  |         |         |
|         v         |  |         v         |     |         |         |  |         |         |
|   +-----------+   |  |   +-----------+   |     |         |         |  |         |         |
|   | Transport |   |  |   | Transport |   |     |         |         |  |         |         |
|   +-----+-----+   |  |   +-----+-----+   |     |         |         |  |         |         |
|         |         |  |         |         |     |         |         |  |         |         |
|         v         |  |         v         |     |         |         |  |         |         |
|   +-----------+   |  |   +-----------+   |     |         |         |  |         |         |
|   | NIC Drvr  |   |  |   | NIC Drvr  |   |     |         |         |  |         |         |
|   +-----+-----+   |  |   +-----+-----+   |     |         |         |  |         |         |
|         |         |  |         |         |     |         |         |  |         |         |
+---------+---------+  +---------+---------+     +---------+---------+  +---------+---------+
          |                      |                         |                      |
          v                      v                         v                      v
       [NIC]<----------------->[NIC]                    [RNIC]<---------------->[RNIC]
```

---

## the hardware

You need an RNIC (RDMA Network Interface Card). This isn't your regular NIC. The RNIC implements RDMA protocols in hardware—it knows how to access memory regions, validate permissions, and move data without running the kernel network stack on every packet.

Two common protocol families:

**InfiniBand (IB):** The original RDMA technology. Requires special switches and cabling. Common in HPC clusters and supercomputers. Latencies under 1μs.

**RoCE (RDMA over Converged Ethernet):** Runs over Ethernet. Easier to deploy if you already have Ethernet infrastructure, but in practice you usually end up treating the fabric as "almost lossless" (PFC/ECN/DCB tuning) or performance gets ugly under loss/congestion. Slightly higher latency than InfiniBand, but close enough for most use cases.

There's also iWARP (RDMA over TCP). It avoids the whole "make Ethernet lossless" dance, but it's less common in modern deployments and often doesn't match RoCE/IB latency.

---

## the programming model

RDMA exposes a different mental model than sockets. Instead of `send()` and `recv()`, you work with queues and memory regions. The main abstractions from `libibverbs` (the userspace library from [rdma-core](https://github.com/linux-rdma/rdma-core)):

**Queue Pair (QP):** Your connection to the hardware. A QP has two queues (a send queue and a receive queue). You post work requests to these queues, and the RNIC processes them asynchronously.

```c
struct ibv_qp *ibv_create_qp(struct ibv_pd *pd,
                             struct ibv_qp_init_attr *qp_init_attr);
```

Creating a QP isn't enough. Fresh QPs start in a `RESET` state. You have to walk them through a state machine: `RESET` -> `INIT` -> `RTR` (ready to receive) -> `RTS` (ready to send). Only then can data actually flow. This trips up everyone the first time.

**Completion Queue (CQ):** How you know an operation finished. When the RNIC completes a work request, it posts a completion entry to the CQ. You poll the CQ or wait for an interrupt.

```c
struct ibv_cq *ibv_create_cq(struct ibv_context *context, int cqe,
                             void *cq_context,
                             struct ibv_comp_channel *channel,
                             int comp_vector);
```

Two modes for processing completions:

- **Polling:** Call `ibv_poll_cq` in a loop. Lowest latency, highest throughput, burns CPU.
- **Interrupt-based:** Use `ibv_req_notify_cq` to arm the CQ for events. Lower CPU usage, higher latency.

A single CQ can serve multiple QPs. This is useful for consolidating completion processing.

**Memory Region (MR):** Before the RNIC can touch your memory, you must register it. Registration does two things: pins the memory so it can't be swapped to disk, and gives you keys.

```c
struct ibv_mr *ibv_reg_mr(struct ibv_pd *pd, void *addr,
                          size_t length, int access);
```

The `access` flags matter. `IBV_ACCESS_LOCAL_WRITE` lets the RNIC write into this MR (e.g., receives). `IBV_ACCESS_REMOTE_READ` and `IBV_ACCESS_REMOTE_WRITE` let remote machines access it directly. The registration returns an `lkey` (used by the local RNIC to validate local work requests) and an `rkey` (presented by remote peers). You share the rkey with the other side.

**Protection Domain (PD):** A security boundary. QPs and MRs belong to a PD. The RNIC checks that any operation matches—you can't use an MR with a QP from a different PD. This replaces the safety checks we lost by bypassing the kernel.

```c
struct ibv_pd *ibv_alloc_pd(struct ibv_context *context);
```

---

## one-sided vs two-sided

RDMA supports both:

**Two-sided (send/receive):** Traditional messaging. One side posts a send, the other posts a receive. Both CPUs are aware of the transfer. Simpler to reason about, similar to sockets.

**One-sided (RDMA read/write):** The magic. RDMA_WRITE shoves data into remote memory without the remote CPU knowing. RDMA_READ pulls data out. The remote application keeps running, oblivious. This is how you get no remote CPU involvement in the data movement.

One-sided operations need the remote memory region's address and rkey. You typically exchange these out-of-band during connection setup.

The gotcha: "oblivious" doesn't mean "safe." You're writing memory via DMA, not calling a function on the remote core. You still need a synchronization protocol (and careful ordering) so the remote side knows when it can read that memory.

---

## who validates operations?

If the kernel isn't inspecting every packet, what stops bad memory accesses?

The RNIC hardware takes over. The setup/transfer split:

**Setup (kernel involved):** When you call `ibv_reg_mr`, the kernel tells the hardware exactly which addresses are valid and what permissions apply. The hardware stores this.

**Transfer (kernel asleep):** During data movement, the RNIC checks every operation against those rules. If you mess up local registration/keys/permissions, you'll see local errors like `IBV_WC_LOC_PROT_ERR`. If the remote rejects the request (bad rkey/permissions or invalid remote address), you'll see a remote error like `IBV_WC_REM_ACCESS_ERR` or `IBV_WC_REM_INV_REQ_ERR`.

The hardware also handles corruption. It checks CRCs, drops bad packets, requests retries. You get reliable delivery without TCP's overhead.

---

## where rdma wins

- **Latency-critical paths:** Sub-microsecond matters when you're doing millions of small operations per second.
- **High-throughput bulk transfers:** The CPU isn't the bottleneck when the RNIC handles everything.
- **Memory disaggregation:** Systems like Infiniswap swap pages to remote memory over RDMA. Works because you're paying microseconds over the fabric instead of milliseconds to disk.
- **Distributed databases and KV stores:** RAMCloud, FaRM, and others build on RDMA for fast replication and reads.

## where it hurts

- **Programming complexity:** The queue/completion model is harder than sockets. State machines, pinned memory, keys—there's a lot to get right.
- **Debugging is painful:** Problems manifest as cryptic completion status codes. No tcpdump. Tools are improving but still rough.
- **Hardware cost:** RNICs and InfiniBand switches aren't cheap. RoCE helps if you already have decent Ethernet.
- **Not worth it for large, infrequent transfers:** If you're sending a few big blobs over slow intervals, TCP's overhead is negligible and the simplicity wins.

---

## notes

- Paper: [RDMA is Turing complete, we just did not know it yet!](https://www.usenix.org/system/files/nsdi22-paper-reda_1.pdf)
- RDMA's "zero-copy" means the RNIC reads directly from your user-space buffer. No kernel buffer, no extra copy. The CPU does zero memory-copy work for the transfer itself.
- Memory registration (`ibv_reg_mr`) pins pages via the kernel/driver so the RNIC can DMA safely. Large registrations can hit `ulimit -l` / `RLIMIT_MEMLOCK` limits.
- QP state transitions are required because each state enables different capabilities and checks. You can't skip steps.
- Scatter/gather elements (SGEs) let you describe non-contiguous memory in a single work request. They always point to local memory.
- One application can have multiple QPs and CQs. CQs aren't 1:1 with queues—flexibility for different polling strategies.
- Atomic operations (compare-and-swap, fetch-and-add) are also available for lock-free distributed algorithms.
- libibverbs man pages: [ibv_create_qp](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_create_qp.3), [ibv_reg_mr](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_reg_mr.3), [ibv_create_cq](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_create_cq.3)
