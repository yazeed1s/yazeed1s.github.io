+++
title = "Memory Disaggregation: Decoupling Memory from Compute"
date = 2025-03-23
description = "How modern data centers separate memory pools from compute nodes."
[taxonomies]
tags = ["Memory", "Distributed Systems", "Data Centers", "Hardware"]
+++

---

## Introduction

- What is memory disaggregation?
- Why do we need it? (memory stranding, resource utilization)
- The shift from monolithic servers to disaggregated architectures

---

## The Problem with Traditional Memory

- Memory tied to individual servers
- Stranded memory (unused but unavailable to other nodes)
- Over-provisioning vs under-utilization

---

## How Disaggregation Works

- Separating memory pools from compute pools
- Remote memory access mechanisms
- Latency considerations

---

## Key Technologies

- RDMA (link to rdma.md)
- CXL (link to cxl.md)
- Network fabrics

---

## Trade-offs

- Latency vs utilization
- Complexity vs flexibility
- When disaggregation makes sense

---

## Conclusion

- Current state of adoption
- Future directions

---

## References

- Papers to cite
- Further reading
