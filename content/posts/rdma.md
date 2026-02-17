+++
title = "RDMA: Bypassing the Kernel for Network I/O"
date = 2026-01-16
description = "How RDMA bypasses the kernel on the data path."
[taxonomies]
tags = ["RDMA", "Networking", "Kernel Bypass", "Systems Programming"]
+++

---

RDMA lets one machine read and write another machine's memory. The network card handles it. The remote CPU doesn't even know it happened.

That sounded wrong to me at first. I knew the slogan, "skip the kernel, go fast," but I didn't understand what it meant at the hardware level.

---

## what's wrong with normal networking

When you use TCP, kernel is always involved.

The app calls `send()`, which is a syscall, so execution enters the kernel; your data gets copied from the app buffer to a kernel buffer, TCP runs its state machine (checksum, segmentation, queueing), and eventually the driver sends it to the NIC.

Receive side same thing. NIC gets packet, interrupt, kernel wakes up, copies to socket buffer. App calls `recv()`, another copy into app buffer.

So you end up with two copies, multiple syscalls, and context switches every time, and the CPU stays busy with all of it.

If you're moving big files, it's OK. Overhead doesn't matter much. But if you want millions of small operations per second, like key-value gets, this overhead is too much.

---

## what rdma does different

With RDMA, the network card reads and writes directly to your app memory.

When you send, the NIC reads from your buffer via DMA, and when you receive, it writes into your buffer via DMA, so the kernel is off the fast path and extra copies disappear.

There's also one-sided operations: RDMA_WRITE puts bytes into remote memory and RDMA_READ pulls bytes from it, while the remote CPU keeps running because nobody wakes it up.

First time I saw this it looked strange. You're writing to memory on different machine, through network card, and that machine doesn't know it happened.

> "Doesn't know" means the remote CPU isn't interrupted and doesn't execute any code. But the NIC is still doing DMA over PCIe, which consumes memory bandwidth on the remote machine. At high throughput, one-sided RDMA operations can noticeably affect remote-side performance even though no software runs there.

![TCP vs RDMA](/images/rdma.png)

---

## setup vs data path

OK but the kernel is still there. Just not on data path.

Before you send anything, you have to set things up by opening the device, creating queues, registering memory, and connecting to the remote side; all of this goes through syscalls and kernel checks.

Only after setup does the fast path work, where you post work directly to hardware and poll completions without syscalls for that part.

So the trade is: expensive setup, cheap operations after. Good if you do many operations. Not good for connections that don't last.

---

## queue pairs

RDMA doesn't use sockets. Uses queues instead.

**Queue Pair** is your connection. Has send queue and receive queue. You put work requests there, saying what to do (send this buffer, read from that address). NIC processes them when it can.

**Completion Queue** is how you know things finished. NIC puts entries. You poll or wait.

The annoying thing is that queue pairs start in RESET state and must move through RESET → INIT → RTR → RTS; if you miss one transition, nothing works and you usually don't get a useful error, which took me a while to learn the first time.

---

## memory registration

Before NIC can touch your memory, you register it.

What registration does:

- Pins the pages. Memory can't go to swap. Physical addresses stay valid because hardware will DMA there.
- Builds translation in NIC. Hardware needs to know where virtual address X is in physical memory.
- Gives you keys. lkey for your own ops, rkey to share with remote. They need your rkey to access your memory.

Registration is a syscall. This is where kernel checks permissions.

---

## who checks if not kernel

This part confused me for a while.

With normal networking kernel validates everything. Bad pointer? SIGSEGV. Wrong permission? Error. Kernel is the one checking.

But RDMA kernel is not in data path. So how bad accesses get stopped?

Answer is hardware.

When you register memory, kernel tells the NIC: these addresses valid, these permissions, this protection domain. NIC stores all this in its memory protection tables.

Then during transfers NIC checks every operation:

- Is address in registered region?
- Permissions OK?
- Key matches?

If something wrong, operation fails. Error shows in completion queue. Not SIGSEGV because NIC caught it, not CPU.

Hardware does what kernel would do, just at wire speed.

---

## protection domains

You can't access anyone's memory.

**Protection Domain** is security boundary. When you make queue pair and register memory, you put them in a PD. Operations only work on memory in same PD.

This is like kernel process isolation but for RDMA. Different apps get different PDs.

---

## one-sided and two-sided

Two kinds of operations.

**Two-sided.** Both sides do something. Receiver posts buffer first. Sender posts send. Both CPUs involved.

**One-sided.** Only you do something. RDMA_WRITE pushes data to remote memory. RDMA_READ pulls data. Remote CPU not involved. Not even aware.

One-sided is where RDMA is powerful. But also more work. If remote app needs to know you wrote, you have to tell it. Usually you write a flag that it polls. Or use atomics. Synchronization is your problem to solve.

---

## atomics

There are atomic operations:

- Compare-and-swap
- Fetch-and-add

They run at remote memory, atomically. Remote CPU not involved.

Good for locks, counters. But slower than regular read/write. And some NICs implement in firmware so even slower. Depends on card.

---

## the hardware

You need special NIC. Normal ones don't do RDMA.

**InfiniBand.** The original. Needs its own switches and cables. Very low latency, under microsecond. HPC clusters use this.

**RoCE.** RDMA over Ethernet. Works on regular switches. But Ethernet drops packets and RDMA really doesn't like that. So you configure switches for "lossless" mode. Priority flow control and so on. It gets complicated.

**iWARP.** RDMA over TCP, which is the most compatible option, but TCP adds latency.

I think most datacenters use RoCE v2 now.

---

## some numbers

| What            | How long |
| --------------- | -------- |
| TCP round trip  | 10-50 μs |
| RDMA round trip | 1-5 μs   |
| Local memory    | ~100 ns  |

RDMA is maybe 10x faster than TCP. But still 10x slower than local RAM. This is important when you think about memory disaggregation. You're replacing local memory with remote. Microseconds add up.

---

## what's hard about it

Debugging is not fun. Problems are completion queue errors with codes you have to look up. No tcpdump. When something breaks you look at hardware counters and guess.

Before RDMA works, both sides exchange info. Queue pair numbers, memory keys, addresses. Usually you do this over TCP first. Extra complexity.

Registered memory is pinned. Big registrations hit ulimit.

For two-sided, receiver must post buffers before sender sends. If receiver runs out of buffers, sender operations fail.

---

## notes

- Verbs API is standard interface. libibverbs on Linux.
- Watch `ulimit -l` for locked memory limit.
- One app can have many queue pairs and completion queues. Common pattern is one QP per thread.
- Atomic ops: compare-and-swap, fetch-and-add. Support varies by hardware.
- Paper: [RDMA is Turing complete](https://www.usenix.org/system/files/nsdi22-paper-reda_1.pdf) (yes really)
- Man pages: [ibv_create_qp](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_create_qp.3), [ibv_reg_mr](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_reg_mr.3), [ibv_post_send](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_post_send.3)
- Nvidia docs: [RDMA Aware Networks Programming User Manual](https://docs.nvidia.com/networking/display/RDMAAwareProgrammingv17)
