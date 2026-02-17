+++
title = "Whole-System Persistence"
[extra]
[[extra.authors]]
name = "Darren Mai (Blogger)"
[[extra.authors]]
name = "Isaac Lonergan (Presenter)"
[[extra.authors]]
name = "Mykyta \"Nick\" Syntsia"
[[extra.authors]]
name = "Sam Shaaban"
[[extra.authors]]
name = "Shubhangi Pandey"
[[extra.authors]]
name = "Nat Rurka"
[[extra.authors]]
name = "Adam Bobich (Scribe)"
+++

# Introduction
Databases are oftentimes entirely stored in memory to achieve high throughput and low latency. When power fails however, recovery can last minutes for a single server or hours for a full cluster. Whole-System Persistence suggests an alternative solution. Since there is no distinction between persistent and volatile objects, you can restore the entire state using only in-memory objects. This post will go over a 2012 paper by two researchers at Microsoft Research, Cambridge on the afforementioned concept.

# Background

## Non-Volatile Main Memory (NVRAM)
- The paper assumes servers use NVDIMMs:
    - DRAM backed by flash and ultracapacitors.
    - During normal operation, behaves exactly like DRAM.
    - On power loss, stored capacitor energy copies DRAM contents to flash.
- From the CPU’s perspective, memory simply survives crashes.

## Traditional Persistence Models
Before this work, two dominant approaches existed:
### Block-Based Persistence
- Applications serialize objects to files or databases.
- Data duplication between memory and storage.
- System call overhead and explicit durability management.

### Persistent Heaps
Persistent heaps are more efficient than block-based systems, but still incur heavy runtime cost due to synchronous cache flushes.
- Certain objects are marked persistent.
- Updates use transactional logging.
- Cache lines must be flushed explicitly to guarantee durability.
- Main overhead: synchronous cache flushes during execution.

## The Bottleneck
On modern processors, writes sit in CPU caches. To make data durable, systems must explicitly flush cache lines to memory. These flushes are slow and serialize execution. In persistent heaps, they occur on every commit. This is the core performance problem the paper addresses.

# Whole-System Persistence (WSP)

## Core Idea
- All main memory is persistent.
- After a power failure:
    - Heap, stack, registers, and OS structures are restored.
- Applications are unaware that a crash occurred.
- No persistent-object APIs or annotations required.

## Flush-on-Commit vs. Flush-on-Fail

### Flush-on-Commit
- Every transaction:
    - Flush modified cache lines.
    - Ensure ordering and durability.
- High runtime overhead.

### Flush-on-Fail (WSP)
- During normal execution:
    - No flushing at all.
- When power failure is detected:
    - CPU registers are saved.
    - All caches are flushed.
    - Processors halt.
    - NVDIMMs copy memory contents to flash using residual PSU energy.

### Key Insights
- Standard power supplies retain 10–400 ms of residual energy after power loss.
- Flushing CPU state requires less than 5 ms.
- Persistence overhead moves completely off the critical execution path.

# Architecture Considerations

## CPU and Memory State
- Registers and cache state must be flushed quickly.
- Hardware support ensures ordering and completeness.
- Memory image is preserved exactly at the moment of failure.

## Device State
- Devices (NICs, disks, GPUs) are harder to persist.
- Transitioning devices to safe states can exceed the residual energy window.
- Proposed solutions:
    - Reinitialize devices on resume.
    - Use virtualization so VMs resume while the host OS handles device reset.

## Failure Model
NVRAM is useful for recovery from crash failures such as power outages. It does not protect against:
- Software bugs
- Logical corruption
- Application-level errors

# Performance Results

## Key Findings
- 1.6×–13× runtime speedup over persistent heaps.
- Up to 13× improvement for update-heavy workloads.
- Even read-heavy workloads benefit significantly.
- Time to save processor cache to NVRAM is under 5ms in every scenario.

# What this means
The results show that getting rid of flush-on-commit yields large gains, and that flush-on-fail has better runtime performance.
- Positions persistence as a system-level property, not an application-level feature.
- Removes the volatile vs. persistent object distinction.

## Strengths
- Extremely simple programming model.
- Strong empirical validation (measured PSU energy and flush times).
- Backward compatibility.

## Weaknesses
- Device handling is complex and underdeveloped.
- Assumes full NVRAM deployment.
- Limited distributed-systems evaluation.
- Only addresses crash failures.

# Class Discussion
- We discussed the tradeoff between cost and benefit.
    - The paper argues that adding capacitors for NVDIMM-style backup is relatively inexpensive.
    - However, the practical use case may be niche enough that large-scale adoption is still hard to justify.
- We questioned whether the physical size of the added capacitor hardware could interfere with airflow and cooling.
    - The paper does not address potential ventilation or thermal constraints.
- The paper was written in 2012, targeting DDR2/DDR3-era systems and older CPUs.
    - The authors measured that there was sufficient residual energy time to flush state.
    - We questioned whether this assumption still holds today.
        - Modern servers contain far more memory.
        - Flushing significantly larger memory footprints within the same time window may be more challenging.
- While the core idea remains elegant, its feasibility on today’s high-capacity systems may require updated hardware validation.

# AI Disclosure
Used ChatGPT to summarize paper and get notes. Formatting for github was also done with ChatGPT.
