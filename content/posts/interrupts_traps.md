+++
title = "Interrupts, Traps, and the Kernel Boundary"
date = 2025-02-18
description = "How the CPU switches between user and kernel mode, and what triggers those switches."
[taxonomies]
tags = ["OS", "CPU", "Linux"]
+++

Your app might call a syscall, a packet might arrive on the network card, or the timer might fire; all of these interrupt normal execution, but they're not the same thing. The terminology gets confusing because people use interrupt, trap, and exception loosely.

## user space vs kernel space

First the basics: modern CPUs have privilege levels called rings on x86, where ring 0 is the most privileged (kernel) and ring 3 is least privileged (user applications).

Your application runs in ring 3 and can execute normal instructions, access its own memory, and do math, but it cannot access hardware directly, read/write arbitrary memory addresses, or execute privileged instructions (like changing page tables or disabling interrupts). The kernel runs in ring 0 and can do all of those things.

When you call `read()` to read from a file, your code can't just talk to the disk controller, it has to ask the kernel. The CPU has to switch from ring 3 to ring 0, do the privileged work, then switch back.

This switch is the kernel boundary, and crossing it costs you: register save/restore, privilege change, and sometimes cache disruption. That's why syscalls aren't free.

## interrupts

An interrupt is a signal from hardware that says "stop what you're doing, I need attention." Examples are a keyboard telling you a key was pressed, a network card saying a packet arrived, a timer saying your time slice is up, or a disk controller saying that read you asked for is done.

Interrupts are asynchronous, they happen whenever the hardware needs attention regardless of what the CPU is currently doing. You could be in the middle of a for loop and suddenly an interrupt fires.

When an interrupt happens, the CPU stops executing the current instruction stream, saves the current state (registers, instruction pointer, flags), looks up the interrupt handler in the Interrupt Descriptor Table (IDT), jumps to that handler (now in ring 0), the handler does its work, and then the handler returns and the CPU restores state and continues where it left off. The key thing is that the currently running process doesn't trigger this, it just happens to it.

## traps

A trap is a synchronous exception triggered by the currently running code, and it's intentional. The main example is syscalls: when you call `read()`, the C library eventually executes a special instruction (`syscall` on x86-64, `int 0x80` on older x86) that deliberately triggers a trap.

```
User code calls read()
  -> libc wrapper
    -> syscall instruction (trap into kernel)
      -> kernel syscall handler
        -> returns to user space
```

The difference from interrupts is that you asked for this, the code executing triggered it, and it happens at a specific point in your instruction stream, not randomly.

Other traps and exceptions include page faults (you accessed memory that isn't mapped, which could be a bug or could be demand paging doing its job), division by zero (arithmetic error), invalid opcode (tried to execute garbage), and breakpoint traps (int 3 for debuggers). Some of these are errors that kill your process (division by zero), and some are handled so execution continues (page fault loads the page and then your load instruction retries).

## the naming confusion

Different sources use these terms differently. Here's how I think about it: **interrupt** means external, async, from hardware. **Trap** means internal, sync, intentional (syscalls). **Exception** means internal, sync, usually an error (page fault, div by zero). **Fault** is an exception that can be corrected (page fault) where the instruction retries. **Abort** is an unrecoverable error.

Some people use "exception" as the umbrella term for everything, some use "interrupt" for everything, and the Intel manual has its own definitions. It's messy. What matters is understanding whether the trigger is external (hardware) or internal (executing code), and whether it's expected (syscall) or unexpected (error).

## the interrupt descriptor table

The CPU needs to know where to jump for each interrupt or exception, and this is stored in the IDT, a table in memory that the kernel sets up at boot. Each entry has a handler address, what privilege level can trigger it, and a gate type (interrupt gate, trap gate).

For hardware interrupts, the entries point to kernel interrupt handlers, and for the syscall trap, it points to the syscall entry point. When an interrupt fires, the CPU uses the interrupt number as an index into the IDT, checks privilege, switches to ring 0 if needed, and jumps to the handler address.

## hardware interrupts in more detail

When a device needs attention, it signals an interrupt request (IRQ), and on modern systems this goes through an interrupt controller (APIC). The kernel has to acknowledge the interrupt, figure out which device caused it, call the right driver's handler, and tell the interrupt controller we're done. Handling needs to be fast because interrupts are disabled (or that IRQ is masked) while you're in the handler, and if you take too long you miss other interrupts.

Linux splits this into top half and bottom half: the **top half** runs in interrupt context, does the minimum work, and schedules the bottom half. The **bottom half** runs later with interrupts enabled and does the real work (softirqs, tasklets, workqueues). For example, a network card interrupt's top half grabs the packet from hardware, queues it, and schedules the bottom half, which then processes the packet up the network stack.

## syscall cost

Crossing the kernel boundary isn't free. You pay for saving and restoring registers, switching stacks (user stack to kernel stack), TLB and cache effects, and Spectre mitigations on modern kernels (KPTI, retpolines).

On a modern system, a syscall might take a few hundred nanoseconds, which doesn't sound like much, but if you're doing thousands per second it adds up. That cost is why people use batching (fewer syscalls, more work per call), io_uring (submit many I/O requests with one syscall), and mmap (access files without read() syscalls).

## notes

- On x86-64, `syscall`/`sysret` are faster than the old `int 0x80` method
- `/proc/interrupts` shows interrupt counts per CPU
- `perf stat` can count context switches and syscalls
- NMI (Non-Maskable Interrupt) can't be disabled, used for profiling and panic
- The timer interrupt is what makes preemptive multitasking work
