+++
title = "Interrupts, Traps, and the Kernel Boundary"
date = 2025-02-18
description = "How the CPU switches between user and kernel mode, and what triggers those switches."
[taxonomies]
tags = ["OS", "CPU", "linux"]
+++

A syscall, a packet arriving on the NIC, and the timer firing all end the same way, the CPU stops running your code and starts running kernel code in ring 0. But they're not the same thing, and people use interrupt, trap, exception and fault almost interchangeably, which confused me for longer than I'd like to admit.

## who caused it

The way I ended up sorting it is by asking who caused the switch.

If hardware caused it, it's an interrupt. The NIC got a packet, the disk finished a read, the timer says your time slice is up, and none of that has anything to do with the instruction you're running, you could be in the middle of a loop and it just happens *to* you, asynchronously.

If your own instruction caused it, it's synchronous, and that's the trap/exception family. Some of them you asked for, like a syscall or `int3` from a debugger, and some you didn't, like a page fault, divide by zero, or an invalid opcode. Inside that family there's another split that actually matters, a fault can be fixed and the instruction retries (page fault loads the page, your load runs again and works), while an abort is unrecoverable and something is very wrong.

So interrupt is external and async, trap is internal and intentional, exception is internal and usually not intentional, and fault is the exception that gets retried. Different books will draw these lines a bit differently and I don't think it's worth fighting over.

## where the CPU jumps

At boot the kernel fills the IDT, a table indexed by vector number, each entry holding a handler address and who's allowed to trigger it. When an interrupt or exception fires the CPU saves the instruction pointer, flags and stack, switches to ring 0 if needed, and jumps to the entry for that vector.

Syscalls on x86-64 are the exception to this btw. The old `int 0x80` went through the IDT like any interrupt, but the `syscall` instruction skips it entirely and jumps to an address the kernel wrote into an MSR at boot, which is part of why it's faster.

## top half, bottom half

Hardware interrupt handlers run with that interrupt line masked (or interrupts off entirely), so if they take long, other devices wait. Linux splits the work, the top half runs right away, does the minimum (grab the packet from the NIC, ack the device) and schedules the rest, and the bottom half (softirqs, tasklets, workqueues) does the actual work later with interrupts enabled. For networking the bottom half is where the packet goes up the whole stack, which is a lot of work to be doing "later".

## what crossing costs

Every crossing means saving registers, switching to the kernel stack, and since Spectre also page table switches (KPTI) and other mitigations, which made syscalls noticeably more expensive overnight. A few hundred nanoseconds per syscall sounds like nothing until you do it hundreds of thousands of times a second. That's the reason io_uring batches I/O into shared rings, and why [mmap](@/posts/mmap.md) is attractive, after the first fault there's no crossing at all. Most of the "fast I/O" ideas I've looked at are just different ways to cross this line less often.

## notes

- `/proc/interrupts` shows interrupt counts per CPU per device.
- `perf stat` counts context switches, `strace -c` gives a quick syscall summary.
- NMIs can't be masked, used for watchdogs and profiling.
- The timer interrupt is what makes preemption possible, without it a busy loop would never give the CPU back.
- Divide by zero becomes `SIGFPE`, which you can technically catch, but there's not much useful to do after.
