# How a GPU Works

> **Level:** Advanced · **Related:** [CPU](cpu.md) · [NPU](npu.md) · [Computer Graphics](../02-graphics/computer-graphics.md) · [Shaders](../02-graphics/shaders.md) · [Ray Tracing](../02-graphics/ray-tracing.md)

## 1. The core idea: throughput over latency

A CPU core is built to make **one thread** finish as fast as possible (big caches, branch prediction, out-of-order execution). A GPU is built to make **tens of thousands of threads** finish as a group as fast as possible. It spends transistors on arithmetic units instead of control logic and **hides memory latency by switching to other threads** instead of caching it away.

| | CPU (desktop) | GPU (high end, 2025) |
|---|---|---|
| Cores | 8–24 complex cores | 100–190 SMs/CUs, ~16–24k "CUDA cores"/ALUs |
| Threads in flight | ~32 | 100,000+ |
| Latency hiding | Caches, OoO, prefetch | Massive multithreading |
| Memory bandwidth | ~100 GB/s (DDR5) | ~1–1.8 TB/s (GDDR7), 3–8 TB/s (HBM3e) |
| FP32 throughput | ~2–4 TFLOPS | ~80–100+ TFLOPS |

## 2. Hierarchy of a modern GPU

```
GPU
├── GPCs / Shader Engines (graphics processing clusters)
│   ├── Raster engine, geometry/primitive units
│   └── SMs (NVIDIA "Streaming Multiprocessor") / CUs, WGPs (AMD) / Xe-cores (Intel)
│       ├── 4 partitions, each with:
│       │   ├── Warp scheduler + dispatch
│       │   ├── Register file (64 KB per partition!)
│       │   ├── 32 FP32/INT32 lanes
│       │   ├── Tensor core (matrix multiply-accumulate)
│       │   └── Load/store & special-function units (sin, rsqrt)
│       ├── L1 cache / Shared memory (~128–256 KB, configurable)
│       ├── Texture units (filtering, addressing)
│       └── RT core (BVH traversal, ray-triangle intersection)
├── L2 cache (tens of MB; AMD "Infinity Cache")
├── Memory controllers → GDDR6X/GDDR7 or HBM
└── Front end: command processor, copy/DMA engines, video encode/decode (NVENC/NVDEC, VCN)
```

## 3. SIMT: how "threads" actually run

GPUs use **SIMT** (Single Instruction, Multiple Threads):

- Threads are grouped into **warps** (NVIDIA, 32 threads) or **wavefronts** (AMD, 32 or 64).
- One instruction is issued for the whole warp; each lane operates on its own registers.
- If threads in a warp take different branches (**divergence**), both paths execute serially with lanes masked off.

```cuda
__global__ void kernel(float* x) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i % 2 == 0) x[i] *= 2.0f;   // half the warp idle here
    else            x[i] += 1.0f;   // other half idle here -> 2x cost
}
```

**Occupancy & latency hiding.** A DRAM access takes ~400–800 cycles. While warp A waits, the scheduler issues from warp B, C, D... every cycle, at zero switching cost because *every warp's registers stay resident* in the huge register file. Occupancy = active warps / max warps; it drops if each thread uses too many registers or too much shared memory.

## 4. The programming model (CUDA / HIP / SYCL / compute shaders)

```
Grid  (whole launch)
 └── Blocks / Workgroups   → scheduled onto one SM, can share memory & sync
      └── Threads / Invocations → grouped into warps
```

A complete vector-add in CUDA C++:

```cpp
#include <cuda_runtime.h>
__global__ void vadd(const float* a, const float* b, float* c, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) c[i] = a[i] + b[i];
}
int main() {
    int n = 1 << 24; size_t bytes = n * sizeof(float);
    float *a, *b, *c;
    cudaMallocManaged(&a, bytes); cudaMallocManaged(&b, bytes); cudaMallocManaged(&c, bytes);
    for (int i = 0; i < n; ++i) { a[i] = 1; b[i] = 2; }
    int threads = 256, blocks = (n + threads - 1) / threads;
    vadd<<<blocks, threads>>>(a, b, c, n);
    cudaDeviceSynchronize();
}
```

The same in Python (CuPy / PyTorch dispatch pre-written kernels):

```python
import torch
a = torch.ones(1 << 24, device="cuda"); b = torch.full_like(a, 2)
c = a + b                      # launches an elementwise CUDA kernel
torch.cuda.synchronize()
```

And in the browser with WebGPU (JavaScript + WGSL):

```js
const shader = `
@group(0) @binding(0) var<storage, read_write> data: array<f32>;
@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) id: vec3u) {
  data[id.x] = data[id.x] * 2.0;
}`;
// device.createComputePipeline(...), pass.dispatchWorkgroups(Math.ceil(n / 64))
```

## 5. Memory: the real bottleneck

| Memory | Scope | Latency | Notes |
|---|---|---|---|
| Registers | Thread | 1 cycle | Spills go to "local" memory (slow) |
| Shared memory / LDS | Block | ~20–30 cycles | Programmer-managed scratchpad, has **bank conflicts** |
| L1 / texture cache | SM | ~30 cycles | |
| L2 | Whole GPU | ~200 cycles | |
| VRAM (GDDR/HBM) | Whole GPU | ~400–800 cycles | Wide bus: 256–512-bit GDDR, 1024-bit per HBM stack |
| System RAM via PCIe | Host | microseconds | PCIe 5.0 x16 ≈ 64 GB/s each way |

**Coalescing:** when the 32 threads of a warp access 32 consecutive 4-byte words, the hardware merges it into one 128-byte transaction. Strided or random access multiplies transactions.

**Roofline model:** a kernel is *memory-bound* if its arithmetic intensity (FLOPs per byte moved) is below `peak FLOPS / peak bandwidth` (~50–100 FLOP/byte on modern GPUs). Most elementwise ops are memory-bound; matrix multiply is compute-bound — which is why tiling into shared memory is the classic optimization.

## 6. The graphics pipeline in hardware

The same SMs run graphics. Fixed-function blocks surround them:

```mermaid
flowchart LR
  CP[Command processor] --> IA[Input assembler]
  IA --> VS[Vertex / Mesh shaders on SMs]
  VS --> CLIP[Clip, cull, viewport]
  CLIP --> RAST[Rasterizer: triangles -> 2x2 pixel quads]
  RAST --> EZ[Early-Z / Hi-Z test]
  EZ --> PS[Pixel shaders on SMs + texture units]
  PS --> ROP[ROPs: depth test, blending, MSAA resolve]
  ROP --> FB[(Framebuffer in VRAM)]
```

- **ROPs** (render output units) do blending and depth writes.
- **Texture units** do bilinear/trilinear/anisotropic filtering in hardware.
- **Tile-based rendering** (mobile GPUs: Apple, Arm Mali, Qualcomm Adreno) bins triangles into screen tiles and renders each tile in on-chip memory to save bandwidth/power.

Details in [Computer Graphics](../02-graphics/computer-graphics.md).

## 7. Specialized units

- **Tensor cores / matrix cores (AMD WMMA/MFMA, Intel XMX):** perform `D = A×B + C` on small tiles (e.g., 16×16) in FP16/BF16/FP8/FP4/INT8 per instruction. This is what powers AI training/inference and [DLSS](../02-graphics/dlss.md).
- **RT cores:** hardware BVH traversal and ray/triangle intersection — see [Ray Tracing](../02-graphics/ray-tracing.md).
- **Optical flow accelerator:** motion vectors between frames for [frame generation](../02-graphics/frame-generation.md).
- **Media engines:** H.264/HEVC/AV1 encode/decode independent of shaders.

## 8. How the CPU talks to the GPU

1. Application calls an API: Direct3D 12, Vulkan, Metal, OpenGL, CUDA.
2. The **user-mode driver** (UMD) validates state and compiles shaders (DXIL/SPIR-V → native ISA) and builds **command buffers**.
3. The **kernel-mode driver** (KMD) manages VRAM, schedules submissions onto hardware queues, handles page tables of the GPU's own MMU.
4. The GPU's **command processor** reads the ring buffer via DMA and executes.

| Windows | Linux |
|---|---|
| WDDM driver model: `dxgkrnl.sys` + vendor KMD (`nvlddmkm.sys`, `amdkmdag.sys`) | DRM/KMS subsystem: `amdgpu`, `i915`/`xe`, `nouveau`, NVIDIA `nvidia.ko` / open kernel modules |
| GPU scheduling: "Hardware-accelerated GPU scheduling" setting | Userspace: Mesa (RADV, ANV, radeonsi), vendor stacks |
| TDR: driver reset if GPU hangs > 2 s | GPU reset via `amdgpu` recovery; `/sys/kernel/debug/dri/` |
| Tools: PIX, Nsight, Radeon GPU Profiler, `dxdiag` | `nvidia-smi`, `radeontop`, `intel_gpu_top`, RenderDoc, `vulkaninfo` |

See [Drivers](../03-os-and-software/drivers.md).

## 9. Performance checklist

1. Keep data on the GPU; PCIe transfers are the #1 beginner bottleneck.
2. Launch enough work (tens of thousands of threads) to fill all SMs.
3. Coalesce memory accesses; use shared memory for reuse.
4. Avoid warp divergence in hot loops.
5. Batch draw calls / kernel launches (each launch costs microseconds of CPU).
6. Use lower precision (FP16/BF16/FP8) where accuracy allows — tensor cores are 8–30× faster.

## Further reading
- NVIDIA CUDA C++ Programming Guide; AMD RDNA/CDNA ISA docs
- Fabian Giesen, *A trip through the Graphics Pipeline 2011* (still the best pipeline walk-through)
- Kirk & Hwu, *Programming Massively Parallel Processors*
