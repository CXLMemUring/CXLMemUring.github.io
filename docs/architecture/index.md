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

As the CXL ecosystem matures, innovations like CXLMemUring will become increasingly important. They not only improve system performance but also provide developers with more flexible programming models, ultimately driving the entire computing industry forward.CXLMemUring: Breaking the Memory Wall with Asynchronous CXL Memory Access

In today's computing landscape, we're facing an increasingly critical challenge - the Memory Wall. Whether it's HPC applications, Deep Learning Recommendation Models (DLRM), or Large Language Model (LLM) training, all are constrained by memory access latency and bandwidth limitations. Today, I want to introduce an innovative solution: CXLMemUring, a hardware-software co-design paradigm for asynchronous and flexible parallel CXL memory pool access.

## CXL Technology: Opening New Doors for Memory Expansion

CXL (Compute Express Link) is an emerging technology that expands memory for both host CPUs and device accelerators through a load/store interface. What sets CXL apart from traditional proprietary standards like NVLink is its extension of memory coherency to the PCIe root complex, making co-design more flexible. You can access memory with coherency using near-device computing capabilities.

## The Core Concept of CXLMemUring

CXLMemUring draws inspiration from Linux's io_uring mechanism. The core idea is to offload synthesized memory operations to CXL endpoints, CXL switches, or cores near the CXL root complex (like Intel DSA) to fetch data, while CPUs or accelerators perform other computations in the background.

When CXL completes data loading:

- Data is placed into L1 cache if capacity permits
- The in-core Reorder Buffer (ROB) is notified via mailbox
- Computation resumes on the previous hardware context

## Hardware-Software Co-design Architecture

### Software Stack

CXLMemUring employs a binary JIT approach similar to Apple Rosetta, seamlessly executing binaries. Key features include:

1. **Forward Analysis**: Translates all remotable memory accesses and pointer accesses to CXL byte-addressable access methods (using MLIR)
2. **Backward Analysis**: Identifies all functions with remote pointers passed, which may be rewritten with native local pointers
3. **Profiling-Guided Optimization**: All functions and labels are marked as profiling-guided points for cost model penalties in timing windows
4. **Dynamic Optimization**: JIT can modify code after labels have been called once to achieve better timing windows

### Hardware Stack

Hardware modifications focus on several key areas:

1. **BOOMv3-based Evaluation Platform**: BOOMv3 was chosen because it's an easily modifiable core
2. **CHI Integration**: For simulating access to CXL switches and other accelerators
3. **Co-processor Design**: Adding co-processors near endpoints to calculate memory requests
4. **In-core Logic**: All memory returns go to L1 cache, avoiding interrupts and only setting ROB metadata to activate requests

## Technical Innovations

1. **Asynchronous Access Pattern**: Adopts an io_uring-like asynchronous memory access approach, greatly improving memory access parallelism
2. **Dynamic Code Optimization**: Uses JIT compiler to dynamically analyze and optimize memory access patterns
3. **Flexible Offloading Points**: Supports compute offloading at multiple locations including CXL endpoints, switches, or DSA
4. **Fine-grained Access**: CXL supports 64-byte fine-grained access, more suitable for C++ object access than RDMA's 4KB pages

## Evaluation Goals

CXLMemUring's evaluation aims to answer these key questions:

1. **Effectiveness of Window Size Capture**: How effectively instruction windows are offloaded
2. **Integration with ROB and MSHR**: Exploring optimal integration design approaches
3. **On-chip Size Comparison**: Whether this approach can save chip area
4. **Programming Model Guidance**: Providing direction for future programming paradigms

## Comparison with Existing Solutions

CXLMemUring offers distinct advantages over existing solutions:

- **vs Data Streaming Accelerator (DSA)**: DSA is currently designed only for single-root CPU and bulk memory loads, requiring driver code and auxiliary data transmission for communication
- **vs Asynchronous RDMA/SmartNIC**: RDMA's 4KB granularity is too large for most C++ objects, unsuitable for pointer chasing and indirect memory reading
- **vs "In-order-core" Asynchronous Memory Unit**: Existing solutions only evaluate on in-order cores without fully considering L2 contention, ROB, and MSHR relationships

## Looking Ahead

CXLMemUring represents a new direction in memory access optimization. In future co-design scenarios, CPUs will serve as hubs for combining DSA requests and executing OLAP operations they excel at, while hardware design will only need co-processors near endpoints to calculate memory requests. This design paradigm promises to provide a scalable solution to the memory wall problem.

As CXL technology continues to mature and become more widespread, we believe innovative designs like CXLMemUring will play an increasingly important role in high-performance computing, AI training, and other domains, helping us break through memory bottlenecks and achieve new leaps in computational performance.

The beauty of CXLMemUring lies in its recognition that memory access patterns are often not statically computable, requiring adaptive, runtime optimization. By combining hardware acceleration with intelligent software management, it offers a path forward in our ongoing battle against the memory wall - one that's both practical and scalable for the demands of modern computing workloads.