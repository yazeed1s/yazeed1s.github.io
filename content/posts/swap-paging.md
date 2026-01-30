+++
title = "OS Swap and Paging: Virtual Memory Fundamentals"
date = 2025-03-23
description = "How operating systems manage virtual memory, page faults, and swapping."
[taxonomies]
tags = ["Operating Systems", "Memory", "Virtual Memory", "Linux", "Paging"]
+++

---

## Introduction

- What is virtual memory?
- Why we need it (isolation, abstraction, overcommit)
- The illusion of infinite memory

---

## Paging Basics

- Pages vs frames
- Page tables
- Virtual to physical address translation
- TLB (Translation Lookaside Buffer)

---

## Page Faults

- What triggers a page fault?
- Major vs minor faults
- Page fault handling flow

---

## Swapping

- What is swap space?
- When does the OS swap?
- Swap algorithms (LRU, clock, etc.)
- Swappiness in Linux

---

## Linux Implementation

- `/proc/meminfo` and swap statistics
- `vm.swappiness` tuning
- Swap files vs swap partitions
- `swapon` / `swapoff`

---

## Performance Considerations

- Swap thrashing
- Why SSD swap is better than HDD
- When to disable swap
- OOM killer

---

## Diagram Ideas

- Address translation flow
- Page fault handling
- Swap in/out process

---

## Conclusion

- Tradeoffs of swap
- Modern alternatives (memory compression, zram, remote swap)

---

## References

- Linux kernel documentation
- Operating systems textbooks
