+++
title = "mmap: What It Does, How to Use It, and When"
date = 2025-11-21
description = "The mmap syscall, its flags, and the situations where it actually makes sense."
[taxonomies]
tags = ["Linux", "C", "Systems Programming", "Memory"]
+++

`mmap` maps a region of virtual address space. That region can be backed by a file, by anonymous memory, or by shared memory. The kernel sets up the page table entries but doesn't necessarily load anything into physical memory until you touch it.

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

Six arguments. The combination of `prot` and `flags` determines what you get.

## protection flags (prot)

These control what you can do with the mapped memory:

- `PROT_READ`: pages can be read
- `PROT_WRITE`: pages can be written
- `PROT_EXEC`: pages can be executed (for JIT compilation or loading code)
- `PROT_NONE`: pages cannot be accessed at all (useful for guard pages)

You can combine them with OR. `PROT_READ | PROT_WRITE` is the common case for data. `PROT_READ | PROT_EXEC` is what the loader uses for `.text` segments.

`PROT_NONE` sounds useless but it's how stack guard pages work. Map a page with no permissions between thread stacks. If a stack overflows into it, the hardware traps immediately instead of silently corrupting the next stack.

## mapping flags

### MAP_SHARED vs MAP_PRIVATE

This is the most important distinction.

`MAP_SHARED` means modifications are visible to other processes that map the same file, and changes are eventually written back to the file. This is actual shared memory. Multiple processes can map the same file with `MAP_SHARED` and see each other's writes.

`MAP_PRIVATE` creates a copy-on-write mapping. You can read the file contents, but writes go to a private copy. The underlying file is never modified. Other processes see the original data.

The dynamic linker uses `MAP_PRIVATE` to load shared libraries. Every process maps the same `.so` file. As long as nobody writes to it, they all share the same physical pages. If a debugger patches a function, only that process gets a private copy of the modified page.

### MAP_ANONYMOUS

No file backing. The mapping is initialized to zero. This is how `malloc` allocates large blocks. When you `malloc(1 << 20)`, glibc usually calls `mmap` with `MAP_ANONYMOUS | MAP_PRIVATE` instead of using `brk`.

When combined with `MAP_SHARED`, you get shared anonymous memory (useful for communication between parent and child processes after `fork`).

```c
// allocate 4MB of zeroed memory, no file involved
void *buf = mmap(NULL, 4 << 20, PROT_READ | PROT_WRITE,
                 MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
```

### MAP_FIXED

Forces the mapping to the exact address you specify in `addr`. If something is already mapped there, it gets silently unmapped. Dangerous, but necessary for some use cases (loading ELF segments at specific addresses, implementing custom allocators with known layouts).

Without `MAP_FIXED`, the `addr` argument is a hint. The kernel can place the mapping wherever it wants.

### MAP_FIXED_NOREPLACE

Same as `MAP_FIXED` but fails with `EEXIST` if the address range is already mapped. Added in Linux 4.17. Safer than `MAP_FIXED` because it won't silently destroy existing mappings.

### MAP_POPULATE

Pre-fault all pages immediately. Normally mmap doesn't load anything until you access it (demand paging). `MAP_POPULATE` tells the kernel to read the entire region into memory upfront.

Useful when you know you'll need the whole mapping and want to avoid page faults during processing. The trade-off is slower setup time and higher initial memory usage.

### MAP_HUGETLB

Use huge pages (2MB or 1GB on x86) instead of regular 4KB pages. Reduces TLB pressure for large mappings. Requires huge pages to be available on the system (configured via `/proc/sys/vm/nr_hugepages` or transparent huge pages).

```c
// map 64MB using 2MB huge pages
void *buf = mmap(NULL, 64 << 20, PROT_READ | PROT_WRITE,
                 MAP_ANONYMOUS | MAP_PRIVATE | MAP_HUGETLB, -1, 0);
```

### MAP_NORESERVE

By default, the kernel checks whether enough swap space exists to back a private writable mapping. `MAP_NORESERVE` skips this check. The mapping may succeed but later writes can fail (SIGBUS) if the system runs out of memory. Only matters when overcommit settings restrict it.

### MAP_LOCKED

Lock pages into physical memory and don't allow them to be swapped out. Similar to calling `mlock` after mmap. Requires `CAP_IPC_LOCK` or sufficient `RLIMIT_MEMLOCK`.

### MAP_GROWSDOWN

Used for stack-like mappings that grow downward. The kernel automatically extends the mapping when the process touches the guard page below it. Used internally for thread stacks.

## companion syscalls

### munmap

Releases a mapping:

```c
int munmap(void *addr, size_t length);
```

After this, accessing the region causes a segfault. The kernel frees the page table entries and (eventually) the physical pages.

### msync

Forces modified pages of a `MAP_SHARED` file mapping back to disk:

```c
int msync(void *addr, size_t length, int flags);
```

Flags are `MS_SYNC` (block until written), `MS_ASYNC` (schedule the write but return immediately), or `MS_INVALIDATE` (invalidate cached copies so other mappings see the latest file content).

Without `msync`, the kernel writes dirty pages back at its own pace. If the system crashes before that, changes can be lost.

### mprotect

Changes protection on an existing mapping:

```c
int mprotect(void *addr, size_t len, int prot);
```

Common use: allocate with `PROT_NONE`, then upgrade to `PROT_READ | PROT_WRITE` when needed. JIT compilers use this to toggle between writable (fill code buffer) and executable (run the code).

### madvise

Hints to the kernel about expected access patterns:

- `MADV_SEQUENTIAL`: will access pages sequentially (kernel can prefetch aggressively)
- `MADV_RANDOM`: will access pages randomly (kernel should not prefetch)
- `MADV_WILLNEED`: will need these pages soon (start loading them)
- `MADV_DONTNEED`: won't need these pages soon (can discard them, re-reading from file or re-zeroing anonymous pages)
- `MADV_FREE`: pages can be reclaimed if memory is needed, but keep them if possible (lazy reclaim, Linux 4.5+)
- `MADV_HUGEPAGE`: enable transparent huge pages for this region
- `MADV_NOHUGEPAGE`: disable transparent huge pages

These are hints, not commands. The kernel can ignore them. But in practice they make a measurable difference for large sequential reads (prefetch) and memory-intensive applications that want to release memory without unmapping (MADV_DONTNEED).

## when mmap makes sense

**Reading large files sequentially or with known access patterns.** Map the file, access it through pointers. The kernel handles paging. For read-only workloads where the file fits in memory, this is simple and efficient.

**Shared memory between processes.** Map the same file (or anonymous region) with `MAP_SHARED` in multiple processes. Writes are visible across processes. No copies, no pipes, no serialization. Used for high-performance IPC.

**Large allocations.** glibc uses mmap for allocations above a threshold (typically 128KB). Anonymous private mappings are simpler to manage than growing the heap with `brk` because they can be returned to the OS individually with `munmap`.

**Loading executables and shared libraries.** The dynamic linker maps `.text` (read + execute), `.rodata` (read-only), and `.data` (read + write, copy-on-write) segments from ELF files. This is the most widespread use of mmap and you never call it directly.

**Guard pages and memory protection.** Map regions with `PROT_NONE` to catch overflows. Change protection dynamically with `mprotect`. JIT compilers write code into RW pages, then flip them to RX before execution.

## when mmap is the wrong tool

**Database buffer management.** [Covered in a separate post](@/posts/mmap_databases.md). The OS controls eviction and write-back, which conflicts with WAL ordering and application-level caching.

**Small, frequent, short-lived allocations.** mmap has overhead per call (kernel entry, VMA creation, page table setup). `malloc` with arena-based allocation is better for small objects.

**When you need precise control over I/O scheduling.** mmap gives you demand paging. If you need async I/O, prefetching at specific offsets, or I/O prioritization, use `read`/`write` with `io_uring` or `aio`.

## notes

- `mmap` returns `MAP_FAILED` (which is `(void *)-1`), not `NULL`, on failure. Check against the wrong value and you miss errors.
- You can remap an existing mapping with `mremap` to grow or shrink it without copying. glibc's `realloc` uses this for mmap'd allocations.
- `/proc/[pid]/maps` shows all current mappings for a process. Each line shows the address range, permissions, offset, device, inode, and pathname.
- Mappings are inherited across `fork`. The child gets the same mappings. With `MAP_PRIVATE`, both parent and child share physical pages until one of them writes (copy-on-write). With `MAP_SHARED`, they share the actual pages.
- `MAP_ANONYMOUS` allocations from mmap are always zero-initialized by the kernel (for security). `malloc` makes no such guarantee.
