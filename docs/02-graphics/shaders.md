# How Shaders Work

> **Level:** Advanced · **Related:** [Computer Graphics](computer-graphics.md) · [GPU](../01-hardware/gpu.md) · [Ray Tracing](ray-tracing.md) · [Compilation](../03-os-and-software/code-compilation.md)

## 1. What a shader is

A **shader** is a small program that runs on the GPU **once per element** — per vertex, per pixel, per ray, per compute thread — in massively parallel fashion. The name comes from its original purpose (computing *shading*), but today shaders do all programmable GPU work.

Key properties:
- **Stateless per invocation**: each invocation sees its own inputs; no knowledge of neighbors (except via explicit shared memory/wave ops).
- **SIMT execution**: 32/64 invocations run in lockstep in a warp/wave (see [GPU](../01-hardware/gpu.md)).
- **No recursion** (except limited in ray tracing), no heap, no system calls.

## 2. Shader stages

| Stage | Runs per | Input → Output | Typical work |
|---|---|---|---|
| **Vertex** | Vertex | Object attributes → clip-space position + varyings | Transform, skinning |
| **Hull / Domain** (Tess. control / eval) | Patch / generated vertex | Patches → subdivided vertices | Terrain LOD, displacement |
| **Geometry** | Primitive | Triangle → 0..N triangles | Rarely used (slow) |
| **Task/Amplification + Mesh** | Workgroup | Meshlets → triangles | Modern GPU-driven geometry, culling |
| **Fragment / Pixel** | Covered sample/pixel | Interpolated varyings → color(s), depth | Materials, lighting |
| **Compute** | Thread in a grid | Buffers/textures → buffers/textures | Physics, post-FX, culling, AI |
| **Ray gen / closest-hit / any-hit / miss / intersection** | Ray | Ray → payload | [Ray tracing](ray-tracing.md) |

## 3. Shading languages & toolchain

| Language | API | Compiles to |
|---|---|---|
| **HLSL** | Direct3D, (Vulkan via DXC) | DXIL (DX12) / DXBC (DX11) / SPIR-V |
| **GLSL** | OpenGL, Vulkan | SPIR-V (via glslang) or driver-compiled |
| **WGSL** | WebGPU | Translated by the browser (Tint/Naga) to HLSL/MSL/SPIR-V |
| **MSL** | Metal | AIR |
| **Slang** | Multi-target | HLSL/GLSL/SPIR-V/CUDA/MSL |

```mermaid
flowchart LR
  SRC[HLSL / GLSL source] --> FE[Front-end compiler<br/>DXC / glslang]
  FE --> IR[Portable IR<br/>DXIL / SPIR-V]
  IR --> DRV[Driver JIT compiler<br/>at pipeline creation]
  DRV --> ISA[Native GPU ISA<br/>e.g. RDNA, SASS]
  ISA --> CACHE[(Shader cache on disk)]
```

The **driver compiles IR → native ISA** at pipeline creation. This is the "compiling shaders" step in games and the source of **shader-compilation stutter** when done lazily during gameplay. Mitigations: precompile at load, pipeline caches (`VkPipelineCache`, D3D12 PSO libraries), Steam's shader pre-caching, DX12 "Advanced Shader Delivery". Cache locations: Windows `%LOCALAPPDATA%\NVIDIA\DXCache` / `AMD\DxcCache`; Linux Mesa `~/.cache/mesa_shader_cache`.

## 4. Anatomy of a vertex + fragment shader (GLSL)

```glsl
// ---------- vertex.glsl ----------
#version 450
layout(location = 0) in vec3 inPos;
layout(location = 1) in vec3 inNormal;
layout(location = 2) in vec2 inUV;

layout(set = 0, binding = 0) uniform Camera { mat4 view; mat4 proj; } cam;
layout(push_constant) uniform Push { mat4 model; } pc;

layout(location = 0) out vec3 vWorldPos;   // "varyings": interpolated by rasterizer
layout(location = 1) out vec3 vNormal;
layout(location = 2) out vec2 vUV;

void main() {
    vec4 world = pc.model * vec4(inPos, 1.0);
    vWorldPos = world.xyz;
    vNormal = mat3(transpose(inverse(pc.model))) * inNormal; // normal matrix
    vUV = inUV;
    gl_Position = cam.proj * cam.view * world;
}
```

```glsl
// ---------- fragment.glsl : PBR (GGX / Cook-Torrance) ----------
#version 450
layout(location = 0) in vec3 vWorldPos;
layout(location = 1) in vec3 vNormal;
layout(location = 2) in vec2 vUV;
layout(set = 1, binding = 0) uniform sampler2D albedoMap;
layout(set = 1, binding = 1) uniform sampler2D ormMap;   // occlusion, roughness, metallic
layout(set = 0, binding = 1) uniform Light { vec3 pos; vec3 color; vec3 camPos; } L;
layout(location = 0) out vec4 outColor;

const float PI = 3.14159265;

float D_GGX(float NdotH, float a) {
    float a2 = a * a, d = NdotH * NdotH * (a2 - 1.0) + 1.0;
    return a2 / (PI * d * d);
}
float G_Smith(float NdotV, float NdotL, float rough) {
    float k = (rough + 1.0); k = k * k / 8.0;
    return (NdotV / (NdotV * (1.0 - k) + k)) * (NdotL / (NdotL * (1.0 - k) + k));
}
vec3 F_Schlick(float cosT, vec3 F0) { return F0 + (1.0 - F0) * pow(1.0 - cosT, 5.0); }

void main() {
    vec3 albedo = texture(albedoMap, vUV).rgb;            // sRGB texture auto-linearized
    vec3 orm = texture(ormMap, vUV).rgb;
    float rough = orm.g, metal = orm.b;

    vec3 N = normalize(vNormal);
    vec3 V = normalize(L.camPos - vWorldPos);
    vec3 Ldir = normalize(L.pos - vWorldPos);
    vec3 H = normalize(V + Ldir);
    float NdotL = max(dot(N, Ldir), 0.0), NdotV = max(dot(N, V), 1e-4);

    vec3 F0 = mix(vec3(0.04), albedo, metal);
    vec3 F = F_Schlick(max(dot(H, V), 0.0), F0);
    float D = D_GGX(max(dot(N, H), 0.0), rough * rough);
    float G = G_Smith(NdotV, NdotL, rough);
    vec3 specular = D * G * F / (4.0 * NdotV * NdotL + 1e-4);
    vec3 kd = (1.0 - F) * (1.0 - metal);

    float dist = length(L.pos - vWorldPos);
    vec3 radiance = L.color / (dist * dist);
    vec3 color = (kd * albedo / PI + specular) * radiance * NdotL;
    color += 0.03 * albedo * orm.r;                      // crude ambient
    outColor = vec4(color, 1.0);                          // tone map in post pass
}
```

## 5. Resources & binding model

Shaders read data through:
- **Uniform / constant buffers**: small, read-only, same for all invocations (matrices, light params).
- **Storage buffers / UAVs / SSBOs**: large read-write arrays.
- **Textures + samplers**: filtered image reads (`texture()`, `Sample()`).
- **Push constants / root constants**: tiny, fastest per-draw values.
- **Bindless** / descriptor indexing: index a huge descriptor heap dynamically (`ResourceDescriptorHeap[i]` in HLSL SM 6.6).

The CPU side (Vulkan/D3D12) describes these via **descriptor sets / root signatures** and bakes shaders + fixed state (blend, depth, formats) into an immutable **Pipeline State Object**.

## 6. Compute shaders (general-purpose)

HLSL compute: a separable blur using **groupshared** memory and barriers:

```hlsl
Texture2D<float4>   Input  : register(t0);
RWTexture2D<float4> Output : register(u0);
groupshared float4 tile[256 + 8];              // 256 threads + 4-pixel apron each side
static const float w[5] = {0.227, 0.194, 0.121, 0.054, 0.016};

[numthreads(256, 1, 1)]
void CSMain(uint3 gid : SV_GroupID, uint3 tid : SV_GroupThreadID, uint3 did : SV_DispatchThreadID) {
    tile[tid.x + 4] = Input[did.xy];
    if (tid.x < 4) {                             // load aprons
        tile[tid.x] = Input[int2(max(int(did.x) - 4, 0), did.y)];
        tile[tid.x + 260] = Input[int2(did.x + 256, did.y)];
    }
    GroupMemoryBarrierWithGroupSync();           // wait until whole tile is loaded
    float4 sum = tile[tid.x + 4] * w[0];
    [unroll] for (int i = 1; i < 5; ++i)
        sum += (tile[tid.x + 4 - i] + tile[tid.x + 4 + i]) * w[i];
    Output[did.xy] = sum;
}
```

**Wave intrinsics** (`WaveActiveSum`, `WavePrefixSum`, `subgroupBallot` in GLSL) let lanes of a warp communicate without shared memory — essential for fast reductions, compaction and culling.

## 7. Performance model for shader authors

1. **ALU vs memory**: texture fetches cost hundreds of cycles; the GPU hides them only if occupancy is high.
2. **Register pressure**: more registers per thread → fewer warps resident → worse latency hiding.
3. **Divergence**: `if` on per-pixel data executes both sides for the warp; use `[branch]`/`[flatten]` knowingly.
4. **Precision**: `half`/`min16float`/`mediump` doubles ALU throughput on many GPUs.
5. **Overdraw**: expensive pixel shaders on hidden pixels — use depth pre-pass or visibility buffer.
6. **Uber-shaders vs permutations**: `#define` variants explode compile counts; dynamic branching on uniforms is cheap (coherent).
7. **Derivatives** (`ddx/ddy`, `dFdx`) work because pixels run in 2×2 quads — also why tiny triangles waste shading (helper lanes).

## 8. Shader-like programming on the CPU (for intuition)

A "fragment shader" in plain Python (NumPy vectorizes it across all pixels, similar in spirit to SIMT):

```python
import numpy as np
W, H = 640, 360
y, x = np.mgrid[0:H, 0:W]
uv_x, uv_y = (x - W / 2) / H, (y - H / 2) / H          # normalized coords
d = np.sqrt(uv_x**2 + uv_y**2)
color = 0.5 + 0.5 * np.cos(10 * d - np.array([0, 2, 4])[:, None, None])  # "per-pixel program"
img = (np.clip(color, 0, 1).transpose(1, 2, 0) * 255).astype(np.uint8)
```

The equivalent WGSL fragment shader:

```wgsl
@fragment
fn fs(@builtin(position) p: vec4f) -> @location(0) vec4f {
  let uv = (p.xy - vec2f(320.0, 180.0)) / 360.0;
  let d = length(uv);
  return vec4f(0.5 + 0.5 * cos(10.0 * d - vec3f(0.0, 2.0, 4.0)), 1.0);
}
```

(ShaderToy is an excellent playground for this style.)

## 9. Debugging

- **RenderDoc** (Windows/Linux): capture a frame, inspect every draw, step through pixel shader.
- **PIX** (Windows D3D12), **Nsight Graphics**, **Radeon GPU Analyzer** (view generated ISA, register usage).
- Output debug colors (`outColor = vec4(N * 0.5 + 0.5, 1)`) — the "printf" of shaders; `printf` exists in Vulkan via `GL_EXT_debug_printf`.

## Further reading
- *The Book of Shaders* (thebookofshaders.com); Inigo Quilez articles (iquilezles.org)
- Microsoft HLSL Shader Model 6.x docs; Khronos GLSL & SPIR-V specs; W3C WGSL spec
- Brian Karis, *Real Shading in Unreal Engine 4* (SIGGRAPH 2013)
