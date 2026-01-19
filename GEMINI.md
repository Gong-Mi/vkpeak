# Project Context: vkpeak

## Project Overview
**vkpeak** is a synthetic benchmarking tool designed to measure the peak capabilities of Vulkan-enabled devices (GPUs). Unlike real-world application benchmarks, `vkpeak` focuses on theoretical peak metrics such as GFLOPS (floating-point operations per second), GIOPS (integer operations per second), and memory bandwidth. It utilizes vector operations heavily to saturate the hardware.

The project relies on the **ncnn** framework (specifically its Vulkan backend) to manage compute shaders and GPU interaction.

## Key Technologies
*   **Language:** C++11
*   **Graphics API:** Vulkan (via `ncnn` abstraction)
*   **Build System:** CMake
*   **Dependencies:** `ncnn` (included as a git submodule)

## Directory Structure & Key Files
*   `vkpeak.cpp`: The core source file. It contains the main entry point and defines the GLSL compute shaders used for benchmarking (embedded as string literals). It iterates through various data types (fp32, fp16, int32, etc.) and vector sizes to test performance.
*   `CMakeLists.txt`: The CMake build configuration. It handles building the `ncnn` dependency (with specific optimizations disabled to keep the build minimal) and links it to the `vkpeak` executable.
*   `check_vulkan_extensions.cpp`: A utility source file likely used to inspect available Vulkan extensions on the device.
*   `ncnn/`: Submodule directory containing the ncnn framework.

## Build & Run Instructions

### Prerequisites
*   C++ compiler (supporting C++11)
*   CMake (3.10+)
*   Vulkan SDK
*   Git (for submodules)

### Building from Source
1.  **Clone and prepare submodules:**
    ```bash
    git submodule update --init --recursive
    ```

2.  **Configure and Build:**
    ```bash
    mkdir build
    cd build
    cmake ..
    cmake --build . -j 4
    ```

### Running the Benchmark
Execute the binary from the build directory:
```bash
./vkpeak
```
To target a specific device (by ID):
```bash
./vkpeak 0
```

## Benchmark Metrics Explanation
The tool breaks down performance into specific data types and execution patterns. Understanding these patterns is crucial for **AI Operator Matching**—selecting the optimal kernel implementation (scalar vs. vector, instruction scheduling) for a specific GPU architecture to maximize inference performance.

### FP16 Variant Breakdown
These metrics specifically test different strategies for saturating the GPU's execution units:

| Metric | Pattern | Description | AI Optimization Relevance |
| :--- | :--- | :--- | :--- |
| **fp16-dual** | **Scalar x2** | Uses **scalar** instructions (`half`) with **2 independent accumulation chains** per loop. | Tests the GPU's ability to hide instruction latency via Instruction-Level Parallelism (ILP). Useful for architectures where vectorization is weak or register pressure prevents vector usage. |
| **fp16-vec2** | **Vec2** | Uses **vector-2** instructions (`half2`) with a **single accumulation chain**. | Measures standard SIMD throughput. Indicates the baseline performance for vectorized kernels. |
| **fp16-vec2-x2** | **Vec2 x2** | Uses **vector-2** instructions (`half2`) with **2 independent accumulation chains**. | **The theoretical peak target.** Combines SIMD (spatial) and ILP (temporal) parallelism. If an AI operator (e.g., convolution, matmul) can be written to match this pattern without hitting register limits, it will achieve maximum hardware utilization. |

**Performance Diagnostic Guide:**
*   **`vec2` >> `dual`**: The architecture heavily relies on SIMD (Vector units) for performance. AI kernels *must* be vectorized.
*   **`vec2-x2` > `vec2`**: The architecture has high instruction latency. AI kernels need "loop unrolling" (computing multiple results per thread) to hide latency and saturate the pipeline.
*   **`vec2-x2` < `vec2`**: Register pressure is too high. Complex kernels should avoid aggressive unrolling to prevent occupancy drops.

## Verification & Effectiveness Guide
The user may ask: *"How do I know this compute power is effective and not just calculating garbage?"*

`vkpeak` assumes hardware stability and correctness. It does **not** perform bit-exact verification of the results against a CPU reference for every run. This is standard for peak-throughput micro-benchmarks.

**To ensure the "Effective Compute Power" for LLM/AI:**
1.  **Sanity Check Scores:** Calculate the theoretical peak: `Clock Speed (GHz) * Core Count * Ops/Cycle`.
    *   If `vkpeak` reports significantly **higher** than theoretical (e.g., >120%), the result is likely invalid (compiler optimization or driver bug).
    *   If `vkpeak` reports **NaN/Inf**, the GPU is unstable (overclocked too high or thermal throttling).
2.  **AI Operator Validation:** `vkpeak` measures potential. Real "effective" power for LLMs must be verified by running unit tests (e.g., `ncnn` layer tests) that compare GPU output `L2 Norm` against CPU/PyTorch reference.
3.  **Silent Data Corruption (SDC):** If you suspect the GPU is calculating "wrong content" despite high GFLOPS, run the `check_vulkan_extensions` tool or standard stress tests (like `GpuTest` or `3DMark`) which include image verification.

## Development Conventions
*   **Shader embedding:** Compute shaders are embedded directly in `vkpeak.cpp` as `static const char` strings.
*   **Dependency Management:** `ncnn` is built as a static library within the project tree; system-wide installation of ncnn is not required/assumed.
*   **Cross-Platform:** The tool is designed to work on Windows, Linux, MacOS (via MoltenVK), and Android.
