+++
title = "Signal Handlers for Cleanup in C Programs"
date = 2023-12-17
description = "When your program crashes, how do you run cleanup code?"
[taxonomies]
tags = ["C", "Signals", "Linux"]
+++

---

In C, you put cleanup code at the end of `main()` - free your allocations, close your files, done. But what if your program crashes before reaching that code?

---

## The Problem

Standard cleanup at the end of main only works if your program exits normally. If it crashes - say, a null pointer dereference - the OS raises SIGSEGV and your program dies immediately. Your cleanup code never runs.

```c
#include <stdio.h>
#include <stdlib.h>

char *buffer = NULL;

int main(void) {
    buffer = malloc(500 * 1024 * 1024); // 500 MB
    if (buffer == NULL) {
        return EXIT_FAILURE;
    }
    printf("Allocated 500 MB\n");

    // oops
    char *invalid_ptr = NULL;
    *invalid_ptr = 'x'; // SIGSEGV here

    // never reached
    printf("About to cleanup...\n");
    free(buffer);
    return 0;
}
```

Output:

```bash
$ gcc crash.c -o crash && ./crash
Allocated 500 MB
Segmentation fault (core dumped)
```

The `free(buffer)` never runs.

## First Thought: atexit()

You might think `atexit()` solves this:

```c
void cleanup(void) {
    free(buffer);
}

int main(void) {
    atexit(cleanup);
    // ...
}
```

But `atexit()` handlers only run on **normal** exit - when main returns or when you call `exit()`. A signal like SIGSEGV bypasses atexit entirely.

## The Solution: Signal Handlers

To run cleanup code on crash, you register a signal handler:

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

char *buffer = NULL;

void cleanup_handler(int sig) {
    // Note: using write() instead of printf() - see below
    const char *msg = "Caught signal, cleaning up...\n";
    write(STDERR_FILENO, msg, 31);

    if (buffer != NULL) {
        free(buffer);
        buffer = NULL;
    }

    _exit(1); // use _exit, not exit()
}

int main(void) {
    // register handler for crash signals
    signal(SIGSEGV, cleanup_handler);
    signal(SIGABRT, cleanup_handler);
    signal(SIGINT, cleanup_handler);  // Ctrl+C
    signal(SIGTERM, cleanup_handler); // kill command

    buffer = malloc(500 * 1024 * 1024);
    if (buffer == NULL) {
        return EXIT_FAILURE;
    }
    printf("Allocated 500 MB\n");

    char *invalid_ptr = NULL;
    *invalid_ptr = 'x'; // triggers SIGSEGV -> cleanup_handler runs

    return 0;
}
```

Now when the program crashes, the handler runs first.

## Important: Async-Signal-Safety

There's a catch. Signal handlers can interrupt your program at any point - even in the middle of malloc or printf. If your handler then calls malloc or printf, you can deadlock or corrupt memory.

Only certain functions are safe to call from signal handlers. These are called "async-signal-safe" functions. The POSIX standard defines the list. Key ones:

- `write()` - safe (printf is NOT safe)
- `_exit()` - safe (exit is NOT safe, it runs atexit handlers)
- `close()`, `unlink()`, `fsync()` - safe

`free()` is technically **not** async-signal-safe. In practice, it usually works because you're about to exit anyway, and modern allocators handle it reasonably. But be aware of this.

A safer pattern is to set a flag and let the main program handle cleanup:

```c
volatile sig_atomic_t got_signal = 0;

void handler(int sig) {
    got_signal = sig;
}

int main(void) {
    signal(SIGINT, handler);

    while (!got_signal) {
        // do work
    }

    // cleanup here, in normal context
    cleanup();
    return 0;
}
```

But for crash signals (SIGSEGV, SIGABRT), you can't return to normal execution - the program is already broken. So you do what cleanup you can in the handler and exit.

## Wait, Doesn't the OS Clean Up Anyway?

Yes. On any modern OS (Linux, macOS, Windows), when your process terminates - normally or not - the kernel reclaims all its resources:

- Heap memory (malloc'd memory) - freed
- File descriptors - closed
- Memory mappings - unmapped

So for plain `malloc()` and regular files, you don't actually need signal handlers for cleanup. The OS handles it.

**Where signal handlers matter:**

1. **Shared resources** - shared memory segments (`shm_open`), semaphores, message queues. These persist beyond process lifetime.
2. **Temp files** - if you want to delete temp files on crash.
3. **External state** - network connections you want to close gracefully, database transactions to rollback.
4. **Custom cleanup** - resetting terminal modes, unlocking files, etc.

For my window manager, I use signal handlers to:

- Unmap windows gracefully
- Restore X11 state
- Close the connection to the X server properly

Regular heap memory? I just let the OS clean it up. It's going to do it anyway.

## Summary

- Cleanup at end of main doesn't run if you crash
- `atexit()` doesn't help - it's for normal exits only
- Signal handlers let you run code on crash
- Be careful about async-signal-safety in handlers
- For regular malloc'd memory, the OS cleans up anyway
- Signal handlers are useful for shared resources and external state

The pattern I use in practice:

```c
volatile sig_atomic_t should_exit = 0;

void signal_handler(int sig) {
    should_exit = 1;
    // minimal cleanup for shared resources only
}

int main(void) {
    signal(SIGINT, signal_handler);
    signal(SIGTERM, signal_handler);

    // main loop checks should_exit
    while (!should_exit) {
        // work
    }

    // full cleanup runs here on graceful exit
    cleanup();
    return 0;
}
```

For SIGSEGV/SIGABRT, I mostly just let it crash and let the OS clean up. Unless there's shared state that needs explicit cleanup, adding a handler for crash signals just complicates things without much benefit.
