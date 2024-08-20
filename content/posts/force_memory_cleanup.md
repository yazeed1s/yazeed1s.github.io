---
title: "Force Memory Cleanup on Crash in C Programs"
date: 2023-12-17
description: "Does the main function return on abnormal exit?"
tags: ["C", "memory", "Linux"]
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: false
ShowPostNavLinks: false
ShowWordCount: true
UseHugoToc: true
---
***
As you might already know.. In C (and other compiled languages as well) the `main` function serves as the entry point for program execution. It controls program execution by directing the calls to whatever code lives inside its body sequentially. Typically, sane programmers craft cleanup code that gets executed at the end of the program to free any allocated resources and give it back to the OS. This is simply done by calling `free()` on heap allocated object(s) before the main routine returns and exits out.
***
## Why End-of-Main Cleanup Isn't Enough
In most cases, placing cleanup code at the end of the main function achieves what we want. However, this approach is guaranteed to work **only** if the program exits **normally** (with EXIT_SUCCESS value).

But what happens when things go wrong? Let's say your program crashes because it tries to dereference a NULL pointer (classic C politics). In this case, the OS will raise a SIGSEGV signal faster than you can say 'segmentation fault'. Your carefully crafted cleanup code at the end of main? It's never going to run.

Lets look at this code for example:
```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

char *buffer = NULL;

int main(void) {
	// allocate 500 MB of memory
	size_t size = 500 * 1024 * 1024; // ~500 MB
	buffer = (char *)malloc(size);
	if (buffer == NULL) {
		printf("Failed to allocate memory");
		return EXIT_FAILURE;
	}
	printf("Successfully allocated 500 MB of memory.\n");
	// do some stuff with *buffer*
	// ...

	// access an invalid pointer
	char *invalid_ptr = NULL;
	printf("Accessing invalid pointer...\n");
	*invalid_ptr = 'x'; // this will cause a segmentation fault
	printf("about to cleanup and return...\n");
	// clean up
	free(buffer);
	return 0;
}
```
Here's what's happening:
- We allocate a large chunk of memory (500 MB) using `malloc()`.
- We then attempt to dereference a NULL pointer, which will cause a segmentation fault.
- The cleanup code (`free(buffer)`) and the `return 0` statement are placed at the end of `main()`.

Now, if we run this code, you might expect to see all the printf statements, including "about to cleanup and return...", but you're wrong. Here's what actually gets printed when I run the program:
```bash
$ gcc signal_crash.c -o out && ./out
Successfully allocated 500 MB of memory.
Accessing invalid pointer...
[1] 117619 segmentation fault (core dumped) ./out
```
See? the program never reaches the line `printf("about to cleanup and return...\n");`. In fact, it never passes the `*invalid_ptr = 'x';` line. Our free statement, which was supposed to cleanup, is never getting called into action because the code crashes with a SIGSEGV signal.
  
## Okay I get it, now how can I force the cleanup code to execute?
Some people think that the use of [atexit()](https://en.cppreference.com/w/c/program/atexit) function provided by `<stdlib.h>` will solve this issue. But the thing is, `atexit()` takes a pointer to a function and then execeute that function when the program exits **normally**. It clearly does not solve our issue.
  
To force memory cleanup when the program crashes, a simple signal handler is needed. Don't panic, It is as simple as adding the following lines to our example.
```C
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

// array of signals we want to handle
int signals[] = {SIGINT, SIGTERM, SIGSEGV, SIGABRT};
char *buffer = NULL;

void cleanup(int sig) {
	printf("Caught signal %d. Cleaning up...\n", sig);
	if (sig == SIGINT) {
		printf("SIGINT received. User interrupted the program.\n");
	} else if (sig == SIGTERM) {
		printf("SIGTERM received. Program is being terminated.\n");
	} else if (sig == SIGSEGV) {
		printf("SIGSEGV received. Segmentation fault occurred.\n");
	} else if (sig == SIGABRT) {
		printf("SIGABRT received. Program execution aborted.\n");
	} else {
		printf("Unexpected signal received.\n");
	}
	// free up buffer
	if (buffer != NULL) {
		free(buffer);
		printf("Freed buffer.\n");
		buffer = NULL;
	}
	printf("Cleanup complete. Exiting program.\n");
	exit(1);
}

void register_signals(void) {
	int n = sizeof(signals) / sizeof(signals[0]);
	for (int i = 0; i < n; i++) {
		signal(signals[i], cleanup);
	}
}

int main(void) {
	// register signal handler
	register_signals();

	// allocate 500 MB of memory
	size_t size = 500 * 1024 * 1024; // ~500 MB
	buffer = (char *)malloc(size);
	if (buffer == NULL) {
	printf("Failed to allocate memory");
	return EXIT_FAILURE;
	}

	printf("Successfully allocated 500 MB of memory.\n");
	// do some stuff with *buffer*
	// ...

	// access an invalid pointer
	char *invalid_ptr = NULL;
	printf("Accessing invalid pointer...\n");
	*invalid_ptr = 'x'; // this will cause a segmentation fault

	return 0;
}

```
Here is what changed:
- I defined an array names `signals` that stores the signal we are interested in (SIGINT, SIGTERM, SIGSEGV, SIGABRT)
- Then I implemented the `cleanup()` function to handle these signals, which prints messages and frees the buffer.
- Then I iterated through the array in `register_signals()` and registered the cleanup function to handle each signal.

When we run this updated code, we should see the cleanup messages printed after the segmentation fault occurs.

Here is the output:
```bash
$ gcc signal_crash.c -o out && ./out
Successfully allocated 500 MB of memory.
Accessing invalid pointer...
Caught signal 11. Cleaning up...
SIGSEGV received. Segmentation fault occurred.
Freed buffer.
Cleanup complete. Exiting program.
```
Now, by implementing this simple signal handling mechanism we are telling the operating system to associate the signal number with the function handler `cleanup()` so that when that signal number is raised (e.g., when the user presses Ctrl+C for SIGINT) our signal handler function will be called. Which also means, our cleanup code will be executed regardless of the program exit circumstances. Goal achieved!!  
***
## But hey doesn't the OS clean up after crashed programs?
That's a great question! The answer isn't as straightforward as we might hope.

In general, modern OSs are pretty good at cleaning up after crashed programs, but it's not 100% guaranteed in all cases. For most typical program crashes, yes, the OS will step in and free up the memory that was allocated to the program and reclaim other resources. But (there's always a but, right?), there are some scenarios where things can get messy due to extremely abnormal terminations.
  
To be honest: I don't care about relying on the OS for cleanup, and you shouldn't either. It's still our responsibility as developers to manage our resources properly and not rely on the OS to bail us out.
***