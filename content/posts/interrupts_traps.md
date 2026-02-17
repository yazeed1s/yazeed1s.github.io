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

Your application runs in ring 3. It can execute normal instructions, access its own memory, do math. But it cannot:

- Access hardware directly
- Read/write arbitrary memory addresses
- Execute privileged instructions (like changing page tables or disabling interrupts)

The kernel runs in ring 0. It can do all of those things.

When you call `read()` to read from a file, your code can't just talk to the disk controller. It has to ask the kernel. The CPU has to switch from ring 3 to ring 0, do the privileged work, then switch back.

This switch is the kernel boundary, and crossing it costs you: register save/restore, privilege change, and sometimes cache disruption. That's why syscalls aren't free.

## interrupts

An interrupt is a signal from hardware that says "stop what you're doing, I need attention."

Examples:

- Keyboard: "a key was pressed"
- Network card: "a packet arrived"
- Timer: "your time slice is up"
- Disk controller: "that read you asked for is done"

Interrupts are asynchronous. They happen whenever the hardware needs attention, regardless of what the CPU is currently doing. You could be in the middle of a for loop and suddenly an interrupt fires.

When an interrupt happens:

1. CPU stops executing current instruction stream
2. Saves current state (registers, instruction pointer, flags)
3. Looks up the interrupt handler in the Interrupt Descriptor Table (IDT)
4. Jumps to that handler (now in ring 0)
5. Handler does its work
6. Handler returns, CPU restores state, continues where it left off

The key thing: the currently running process doesn't trigger this. It just happens to it.

## traps

A trap is a synchronous exception triggered by the currently running code. It's intentional.

The main example: syscalls. When you call `read()`, the C library eventually executes a special instruction (`syscall` on x86-64, `int 0x80` on older x86) that deliberately triggers a trap.

```
User code calls read()
  -> libc wrapper
    -> syscall instruction (trap into kernel)
      -> kernel syscall handler
        -> returns to user space
```

The difference from interrupts: you asked for this. The code executing triggered it. It happens at a specific point in your instruction stream, not randomly.

Other traps/exceptions:

- Page fault — you accessed memory that isn't mapped. Could be a bug, or could be demand paging doing its job.
- Division by zero — arithmetic error
- Invalid opcode — tried to execute garbage
- Breakpoint — debugger trap (int 3)

Some of these are errors (division by zero kills your process). Some are handled and execution continues (page fault loads the page, then your load instruction retries).

## the naming confusion

Different sources use these terms differently. Here's how I think about it:

- **Interrupt** — external, async, from hardware
- **Trap** — internal, sync, intentional (syscalls)
- **Exception** — internal, sync, usually an error (page fault, div by zero)
- **Fault** — exception that can be corrected (page fault) — instruction retries
- **Abort** — unrecoverable error

Some people use "exception" as the umbrella term for everything. Some use "interrupt" for everything. The Intel manual has its own definitions. It's messy.

What matters: understand whether the trigger is external (hardware) or internal (executing code), and whether it's expected (syscall) or unexpected (error).

## the interrupt descriptor table

The CPU needs to know where to jump for each interrupt/exception. This is stored in the IDT, a table in memory. The kernel sets this up at boot.

Each entry has:

- Handler address
- What privilege level can trigger it
- Gate type (interrupt gate, trap gate)

For hardware interrupts, the entries point to kernel interrupt handlers. For the syscall trap, it points to the syscall entry point.

When an interrupt fires, the CPU:

1. Uses the interrupt number as an index into IDT
2. Checks privilege
3. Switches to ring 0 if needed
4. Jumps to the handler address

## hardware interrupts in more detail

When a device needs attention, it signals an interrupt request (IRQ). On modern systems this goes through an interrupt controller (APIC).

The kernel has to:

1. Acknowledge the interrupt
2. Figure out which device caused it
3. Call the right driver's handler
4. Tell the interrupt controller we're done

Handling needs to be fast because interrupts are disabled (or that IRQ is masked) while you're in the handler. If you take too long, you miss other interrupts.

Linux splits this into top half and bottom half:

- **Top half**: runs in interrupt context, does minimum work, schedules bottom half
- **Bottom half**: runs later with interrupts enabled, does the real work (softirqs, tasklets, workqueues)

For example, network card interrupt:

- Top half: grab the packet from hardware, queue it, schedule bottom half
- Bottom half: process the packet up the network stack

## syscall cost

Crossing the kernel boundary isn't free. You pay for:

- Saving/restoring registers
- Switching stacks (user stack -> kernel stack)
- TLB and cache effects
- Spectre mitigations on modern kernels (KPTI, retpolines)

On a modern system, a syscall might take a few hundred nanoseconds. Doesn't sound like much, but if you're doing thousands per second, it adds up.

That cost is why people use:

- Batching (fewer syscalls, more work per call)
- io_uring (submit many I/O requests with one syscall)
- mmap (access files without read() syscalls)

## notes

- On x86-64, `syscall`/`sysret` are faster than the old `int 0x80` method
- `/proc/interrupts` shows interrupt counts per CPU
- `perf stat` can count context switches and syscalls
- NMI (Non-Maskable Interrupt) can't be disabled, used for profiling and panic
- The timer interrupt is what makes preemptive multitasking work
