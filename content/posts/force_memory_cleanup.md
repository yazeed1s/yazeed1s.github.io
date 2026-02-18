+++
title = "Signal Handlers for Cleanup in C Programs"
date = 2023-12-17
description = "When your program crashes, how do you run cleanup code?"
[taxonomies]
tags = ["C", "Signals", "Linux"]
+++

In C, you put cleanup code at the end of `main()` to free your allocations, close your files, and so on. But what if your program crashes before reaching that code?

## the problem

Standard cleanup at the end of main only works if your program exits normally. If it crashes (a null pointer dereference, for example), the OS raises SIGSEGV and your program dies immediately, so your cleanup code never runs.

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

## first thought: atexit()

You might think `atexit()` solves this, where you register a cleanup function and it gets called on exit:

```c
void cleanup(void) {
    free(buffer);
}

int main(void) {
    atexit(cleanup);
    // ...
}
```

But `atexit()` handlers only run on **normal** exit (when main returns or when you call `exit()`), and a signal like SIGSEGV bypasses atexit entirely.

## the solution: signal handlers

To run cleanup code on crash, you register a signal handler that catches the signal before the process dies:

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

char *buffer = NULL;

void cleanup_handler(int sig) {
    (void)sig;
    // Note: using write() instead of printf() - see below
    const char msg[] = "Caught signal, exiting...\n";
    write(STDERR_FILENO, msg, sizeof(msg) - 1);

    // For crash signals, keep the handler minimal. Most cleanup is not
    // async-signal-safe, and the kernel will reclaim process memory anyway.
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

## async-signal-safety

There's an important catch: signal handlers can interrupt your program at any point, even in the middle of malloc or printf. If your handler then calls malloc or printf, you can deadlock or corrupt memory because those functions aren't reentrant.

Only certain functions are safe to call from signal handlers, and POSIX defines the list. The key safe ones are `write()` (printf is NOT safe), `_exit()` (exit is NOT safe because it runs atexit handlers), `close()`, `unlink()`, and `fsync()`. Notably, `free()` is **not** async-signal-safe; it might "work" in a toy program and then deadlock in production because the signal interrupted malloc/free internals.

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

But for crash signals (SIGSEGV, SIGABRT), you can't return to normal execution because the program is already broken, so you do what cleanup you can in the handler and exit.

## wait, doesn't the OS clean up anyway?

Yes. On any modern OS (Linux, macOS, Windows), when your process terminates the kernel reclaims all its resources: heap memory gets freed, file descriptors get closed, and memory mappings get unmapped.

So for plain `malloc()` and regular files, you don't actually need signal handlers for cleanup because the OS handles it. Where signal handlers really matter is shared resources (shared memory segments from `shm_open`, semaphores, message queues) that persist beyond process lifetime, temp files you want to delete on crash, external state like network connections or database transactions you want to close gracefully, and custom cleanup like resetting terminal modes or unlocking files.

For my window manager, I use signal handlers to unmap windows gracefully, restore X11 state, and close the connection to the X server properly. Regular heap memory? I just let the OS clean it up because it's going to do it anyway.

## the pattern I use

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

## notes

- The Summary section from the original was integrated into the prose above
- `signal()` behavior varies by platform, `sigaction()` is more portable and predictable
- You can use `SA_RESETHAND` flag with sigaction to restore default behavior after catching a crash signal once
