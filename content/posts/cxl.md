+++
title = "CXL: Compute Express Link"
date = 2025-03-23
description = "The new interconnect standard for memory expansion and disaggregation."
[taxonomies]
tags = ["CXL", "Memory", "Hardware", "PCIe", "Data Centers"]
+++

---

## Introduction

- What is CXL?
- Why a new interconnect? (PCIe limitations for memory)
- Industry adoption (Intel, AMD, ARM, etc.)

---

## CXL vs Existing Technologies

- CXL vs PCIe
- CXL vs RDMA (link to rdma.md)
- CXL vs NUMA
- Latency and bandwidth comparison

---

## CXL Protocol Types

- CXL.io (PCIe compatibility)
- CXL.cache (cache coherent access)
- CXL.mem (memory access)

---

## CXL Device Types

- Type 1: Accelerators (cache coherent)
- Type 2: Accelerators with memory
- Type 3: Memory expanders

---

## Memory Expansion Use Cases

- Adding memory to existing servers
- Memory pooling
- Tiered memory (fast local + slower CXL)

---

## Software Support

- Linux kernel support
- Memory hotplug
- NUMA integration
- Application transparency

---

## CXL 2.0 and 3.0

- Switching and pooling
- Memory sharing
- What's new in each version

---

## Challenges

- Latency overhead vs local DRAM
- Software ecosystem maturity
- Cost

---

## Conclusion

- CXL's role in memory disaggregation
- Future of data center memory

---

## References

- CXL specification
- Industry whitepapers
- Research papers
