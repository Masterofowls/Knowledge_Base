# How Ray Tracing Works

> **Level:** Advanced · **Related:** [Computer Graphics](computer-graphics.md) · [Shaders](shaders.md) · [GPU](../01-hardware/gpu.md) · [DLSS](dlss.md)

## 1. The idea

Light travels from sources, bounces around, and some reaches the eye. Simulating every photon from lights is wasteful (most never reach the camera), so ray tracing runs it **backwards**: shoot rays *from the camera* through each pixel, find what they hit, and ask "how much light arrives here?" by shooting more rays toward lights and in bounce directions.

```
 camera ●──────ray──────▶ ■ surface hit
        │                  │╲ shadow ray ─▶ ☀ light (blocked? → shadow)
        │                  │ ╲ reflection/bounce ray ─▶ other surfaces / sky
   pixel grid
```

| Variant | What it traces | Result |
|---|---|---|
| Ray casting (Appel 1968) | Primary rays only | Visibility |
| Whitted ray tracing (1980) | + shadow, perfect reflection, refraction rays | Sharp mirrors, glass |
| Distributed ray tracing (Cook 1984) | Many rays with jitter | Soft shadows, DOF, motion blur, glossy |
| **Path tracing** (Kajiya 1986) | Random bounce paths, Monte Carlo | Full global illumination — the physically based standard |

## 2. Ray–primitive intersection

A ray: `P(t) = O + t·D`, `t > 0`.

**Sphere** (center C, radius r): solve `|O + tD − C|² = r²` — a quadratic.

**Triangle** — Möller–Trumbore algorithm (what RT hardware effectively implements):

```cpp
#include <optional>
struct Vec3 { float x, y, z; };
Vec3 sub(Vec3 a, Vec3 b) { return {a.x-b.x, a.y-b.y, a.z-b.z}; }
Vec3 cross(Vec3 a, Vec3 b) { return {a.y*b.z-a.z*b.y, a.z*b.x-a.x*b.z, a.x*b.y-a.y*b.x}; }
float dot(Vec3 a, Vec3 b) { return a.x*b.x + a.y*b.y + a.z*b.z; }

// Returns distance t along ray and barycentrics (u, v) if hit
std::optional<float> intersect(Vec3 O, Vec3 D, Vec3 v0, Vec3 v1, Vec3 v2, float& u, float& v) {
    const float EPS = 1e-7f;
    Vec3 e1 = sub(v1, v0), e2 = sub(v2, v0);
    Vec3 p = cross(D, e2);
    float det = dot(e1, p);
    if (std::abs(det) < EPS) return std::nullopt;      // ray parallel to triangle
    float inv = 1.0f / det;
    Vec3 s = sub(O, v0);
    u = dot(s, p) * inv;            if (u < 0 || u > 1) return std::nullopt;
    Vec3 q = cross(s, e1);
    v = dot(D, q) * inv;            if (v < 0 || u + v > 1) return std::nullopt;
    float t = dot(e2, q) * inv;
    return t > EPS ? std::optional<float>(t) : std::nullopt;
}
```

## 3. Acceleration structures: the BVH

Testing every ray against millions of triangles is impossible (1920×1080 × 10⁷ = 2×10¹³ tests). A **Bounding Volume Hierarchy** organizes triangles into a tree of axis-aligned bounding boxes (AABBs):

```
               [Root AABB]
              /            \
        [AABB L]          [AABB R]
        /      \          /      \
   [leaf:3 tris][...] [...]   [leaf:4 tris]
```

Traversal: test ray vs box (the **slab test**: cheap min/max of `(bounds − O)/D` per axis); descend only into hit children, nearest first; prune when a closer hit is known. Cost becomes ~O(log N).

- **Building**: SAH (Surface Area Heuristic) chooses splits minimizing expected cost = `Σ area(child)/area(parent) × triangles(child)`.
- **Two-level** in DXR/Vulkan RT:
  - **BLAS** (bottom-level): geometry of one mesh, built once (or refitted for deformation).
  - **TLAS** (top-level): instances of BLASes with transforms, rebuilt every frame cheaply.
- Hardware BVHs are wide (4–8 children per node) and compressed.

## 4. Path tracing and Monte Carlo

The [rendering equation](computer-graphics.md#4-lighting-and-materials) integral has no closed form, so we estimate it with random samples:

```
Lo ≈ Le + (1/N) Σ [ fr(ωi, ωo) · Li(ωi) · cosθi / pdf(ωi) ]
```

A single path: hit surface → sample a direction from the BRDF (importance sampling) → recurse → multiply contributions ("throughput") → stop via **Russian roulette**.

Minimal path tracer core (Python-like pseudocode, runnable concept):

```python
def radiance(ray, scene, depth=0):
    hit = scene.intersect(ray)                 # BVH traversal
    if hit is None:
        return scene.sky(ray.dir)
    L = hit.material.emission
    # Next-event estimation: sample a light directly (reduces noise massively)
    light_pt, light_pdf, Le = scene.sample_light(hit.pos)
    if not scene.occluded(hit.pos, light_pt):  # shadow ray
        L += hit.material.brdf(hit, light_pt) * Le * cos_term(hit, light_pt) / light_pdf
    # Indirect bounce
    if depth > 3:                              # Russian roulette
        p = max(hit.material.albedo)
        if random() > p: return L
    else:
        p = 1.0
    wi, pdf = hit.material.sample(hit)         # importance-sample BRDF
    L += hit.material.brdf(hit, wi) * radiance(Ray(hit.pos, wi), scene, depth + 1) \
         * abs(dot(hit.normal, wi)) / (pdf * p)
    return L
```

**Noise** falls as 1/√N — 4× the samples for half the noise. Film renders use thousands of samples per pixel (spp); games get **~0.5–2 spp** and rely on:

- **Importance sampling** + **Multiple Importance Sampling (MIS)**
- **ReSTIR** (Reservoir Spatio-Temporal Importance Resampling): reuse light samples across neighboring pixels and frames — enables millions of lights
- **Denoisers**: spatiotemporal filters (SVGF), NVIDIA NRD, OptiX AI denoiser, Intel OIDN, and DLSS **Ray Reconstruction** (an AI denoiser fused with upscaling — see [DLSS](dlss.md))

## 5. Hardware ray tracing on GPUs

Since NVIDIA Turing (2018), AMD RDNA2 (2020), Intel Arc (2022), Apple M3/A17:

| Unit | Does |
|---|---|
| **RT cores / Ray accelerators** | Ray-box and ray-triangle tests, BVH traversal in fixed function |
| SMs/CUs (shaders) | Ray generation, material shading, denoising |
| Shader Execution Reordering (SER, NVIDIA Ada+) / sorting | Regroups divergent rays by material for coherence |
| Opacity micromaps, displaced micro-meshes | Faster alpha-tested foliage, compressed detail |

(AMD RDNA2/3 accelerate intersection tests but traverse in shaders; RDNA4 adds more traversal hardware.)

## 6. The programming model: DXR / Vulkan Ray Tracing

Shader types (HLSL DXR):

```hlsl
RaytracingAccelerationStructure Scene : register(t0);
RWTexture2D<float4> Output : register(u0);
struct Payload { float3 color; };

[shader("raygeneration")]
void RayGen() {
    uint2 px = DispatchRaysIndex().xy;
    float2 ndc = (px + 0.5) / DispatchRaysDimensions().xy * 2 - 1;
    RayDesc ray;
    ray.Origin = CameraPos;
    ray.Direction = normalize(CameraForward + ndc.x * CameraRight - ndc.y * CameraUp);
    ray.TMin = 0.001; ray.TMax = 1e4;
    Payload p = { float3(0, 0, 0) };
    TraceRay(Scene, RAY_FLAG_NONE, 0xFF, 0, 1, 0, ray, p);   // hardware traversal
    Output[px] = float4(p.color, 1);
}

[shader("closesthit")]
void ClosestHit(inout Payload p, BuiltInTriangleIntersectionAttributes a) {
    float3 bary = float3(1 - a.barycentrics.x - a.barycentrics.y, a.barycentrics);
    p.color = bary;                                         // visualize barycentrics
}

[shader("miss")]
void Miss(inout Payload p) { p.color = float3(0.4, 0.6, 1.0); }  // sky
```

- **Shader Binding Table (SBT)** maps geometry/material → which hit shaders run.
- `any-hit` shaders handle alpha testing; `intersection` shaders handle procedural geometry.
- **Inline ray tracing** (`RayQuery` in DXR 1.1, `rayQueryEXT` in Vulkan) lets any shader (e.g., a compute or pixel shader) trace rays without the SBT — popular for shadows/AO.

APIs: **DirectX Raytracing (DXR)** on Windows (D3D12), **Vulkan** `VK_KHR_ray_tracing_pipeline`/`VK_KHR_ray_query` on Linux/Windows (on Linux via Mesa RADV/ANV or NVIDIA drivers; Proton translates DXR→Vulkan RT), Metal RT, OptiX/CUDA for offline.

## 7. How games use it (hybrid rendering)

| Effect | Rasterized approach | Ray-traced approach |
|---|---|---|
| Shadows | Shadow maps (aliasing, peter-panning) | Shadow rays: accurate, soft contact shadows |
| Reflections | Screen-space (missing off-screen), cube maps | True reflections of anything |
| Ambient occlusion | SSAO | RTAO |
| Global illumination | Baked / probes / SSGI | RTGI, path traced ("Full RT"/"Overdrive" modes) |

Typical frame: rasterize G-buffer → trace rays at quarter/half resolution → denoise → composite → [upscale](dlss.md) → [frame generation](frame-generation.md).

## 8. Performance factors

- **Ray coherence**: primary/shadow rays are coherent; diffuse bounces scatter → cache misses and divergence.
- BVH quality vs build time (dynamic objects need per-frame refit/rebuild).
- Alpha-tested geometry forces any-hit shader invocations (expensive).
- Rays per pixel and max bounce depth trade quality for time.

## Further reading
- Peter Shirley, *Ray Tracing in One Weekend* series (free; build a path tracer in C++)
- Pharr, Jakob, Humphreys, *Physically Based Rendering* (pbr-book.org)
- Microsoft DirectX Raytracing spec (DirectX-Specs on GitHub); Khronos Vulkan RT guide
- Bitterli et al., *Spatiotemporal reservoir resampling (ReSTIR)* (SIGGRAPH 2020)
