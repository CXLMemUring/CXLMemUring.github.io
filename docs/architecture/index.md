---
weight: 3
bookCollapseSection: false
title: "Project architecture"
---

<img width="601" height="171" alt="image" src="https://github.com/user-attachments/assets/48d5cf2f-df0f-4a25-8573-f1e60f05548c" />

# CXLMemUring: Breaking the Memory Wall with Hardware-Software Co-design

## Introduction: Why Do We Need CXLMemUring?

We're living in the era of the Great Memory Wall. Whether you're working with HPC applications, training Deep Learning Recommendation Models (DLRM), or building Large Language Models (LLM), memory latency and bandwidth have become the critical bottlenecks limiting application scalability.

Enter CXL (Compute Express Link) - an emerging technology that promises to expand memory for both host CPUs and device accelerators through a load/store interface. By extending memory coherency to the PCIe root complex, CXL enables flexible co-design patterns where you can access memory with coherency using near-device compute capabilities.

However, traditional memory optimization techniques like ROB, MSHR, read-ahead cache, and TLB don't scale well to CXL memory pools. This is where CXLMemUring comes in - a revolutionary hardware-software co-design paradigm for asynchronous and flexible parallel CXL memory pool access.

## The Core Concept: Memory Access Like IO_uring

The name CXLMemUring draws inspiration from Linux's IO_uring mechanism. Just as IO_uring revolutionized asynchronous I/O operations, CXLMemUring aims to transform how we access memory.

### How It Works

1. **Offload Memory Operations**: Synthesized memory operations are offloaded to CXL endpoints, CXL switches, or cores near the CXL root complex (like Intel DSA)
2. **Asynchronous Execution**: CPUs or accelerators can perform other computations while memory is being loaded
3. **Smart Notification**: When CXL completes data loading, the data is placed into L1 cache (if capacity permits), and the in-core ROB is notified via a mailbox mechanism to resume computation on the previous hardware context

## Technical Architecture: The Art of Hardware-Software Co-design

### Software Stack

CXLMemUring employs a binary JIT compiler approach, similar to Apple Rosetta:

- **Forward Analysis**: Translates all remotable memory accesses and pointer accesses to CXL byte-addressable format using MLIR
- **Backward Analysis**: Identifies all functions with remote pointers that may be rewritten with native local pointers
- **Dynamic Optimization**: Uses profiling-guided techniques to adaptively adjust the offloading window, since access patterns often cannot be statically computed

The system marks functions and labels as profiling-guided points for cost model penalties. During runtime, the JIT can modify code after labels have been called to achieve better timing windows.

### Hardware Stack

The hardware design consists of two main components:

1. In-CPU Async Loading Engine

   :

   - Notifies the CPU to resume previous context when data arrives
   - Implements callbacks through CXL.io requests with an in-core mailbox

2. Near-Endpoint Coprocessor

   :

   - Computes load instruction sequences and simple memory operations
   - Can be deployed near CXL Flash, GPU, or CXL switches
   - Designed to handle the computational aspects of memory requests

## Key Innovations: Beyond Traditional Approaches

### 1. Fine-Grained Access Optimization

Unlike RDMA's 4KB granularity, CXL supports 64-byte access granularity - much better suited for typical C++ object sizes. CXLMemUring leverages this to excel at pointer chasing and indirect memory reading scenarios.

### 2. Flexible Offloading Points

The system supports multiple offloading locations:

- CXL endpoints
- CXL switches
- DSA near the root complex

Each offloading point has a different cost model, allowing the system to choose the optimal approach based on the specific workload.

### 3. Interrupt-Free Design

Rather than using traditional interrupt-driven approaches, CXLMemUring sets ROB metadata to activate requests. This eliminates interrupt overhead and improves system efficiency.

## Evaluation Framework: Proving the Value

CXLMemUring is evaluated using a modified BOOMv3 implementation on FPGA, with CHI for simulating CXL switch and accelerator access. Key evaluation dimensions include:

1. **Window Capture Effectiveness**: How effectively instruction windows are offloaded and how much computation can be done before memory arrives
2. **ROB/MSHR Integration**: Optimal design patterns for integrating with existing memory subsystems
3. **On-chip Area Comparison**: Whether this approach saves chip area
4. **Programming Model Guidance**: Insights for future programming models

## The Vision: Rethinking Memory Access

CXLMemUring represents a paradigm shift in how we think about computation. In this model:

- The CPU becomes a hub for combining DSA requests and executing OLAP operations
- Control flow is offloaded while most memory remains local
- Only minimal memory communication is required
- Hardware design becomes more flexible, with compute capabilities deployed where needed

## Implementation Insights

The evaluation is based on BOOMv3 over FPGA, chosen for its modifiability. The system adds CHI for simulating CXL switches and accelerator access, plus a weaker RISC-V core near the endpoint for code offloading.

For the in-core logic, all memory returns go into L1 cache to ensure they're unique to the SMT core and instantly consumed. The design avoids interrupts, instead setting ROB metadata to activate requests.

## Looking Forward: The Future of Memory Access

As the capacity demands with tolerable latency and bandwidth continue to grow, approaches like CXLMemUring become increasingly critical. The co-design paradigm allows us to:

- Better utilize CXL's coherency capabilities
- Overcome limitations of traditional prefetching approaches
- Enable new programming models that better match modern workload requirements

## Conclusion

CXLMemUring isn't just a technical solution - it's a new way of thinking about memory access in the CXL era. By combining hardware and software innovation, we can break through the memory wall and enable the next generation of high-performance applications.

As the CXL ecosystem matures, innovations like CXLMemUring will become increasingly important. They not only improve system performance but also provide developers with more flexible programming models, ultimately driving the entire computing industry forward.
