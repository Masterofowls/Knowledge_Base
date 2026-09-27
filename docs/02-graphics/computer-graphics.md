# How Computer Graphics Work

> **Level:** Advanced · **Related:** [GPU](../01-hardware/gpu.md) · [Shaders](shaders.md) · [Ray Tracing](ray-tracing.md) · [DLSS](dlss.md) · [Frame Generation](frame-generation.md)

## 1. The problem

Turn a **scene description** (geometry, materials, lights, camera) into a **2D grid of pixel colors**, typically 60–240 times per second. Two families of algorithms solve this:

| | Rasterization | Ray tracing |
|---|---|---|
| Question asked | "For each triangle, which pixels does it cover?" | "For each pixel, what does a ray from the eye hit?" |
| Complexity | ~O(triangles) | ~O(pixels × log(triangles)) with a BVH |
| Strength | Extremely fast, hardware-native | Physically correct shadows, reflections, GI |
| Used by | Every real-time engine | Film/offline; real-time hybrid since 2018 |

Modern games are **hybrid**: rasterize primary visibility, ray trace selected effects, then denoise and upscale with AI.

## 2. Math foundations

**Homogeneous coordinates** let translation, rotation, scale and perspective be 4×4 matrices:

```
p_clip = P · V · M · p_object
   M: model (object → world)
   V: view (world → camera)   = inverse(camera transform)
   P: projection (camera → clip space)
```

Perspective projection (right-handed, depth to [0,1] as in D3D/Vulkan):

```
f = 1 / tan(fovY/2)
P = | f/aspect  0      0              0           |
    | 0         f      0              0           |
    | 0         0      far/(near-far) near*far/(near-far) |
    | 0         0     -1              0           |
```

After multiplying, the GPU divides by `w` (**perspective divide**) → Normalized Device Coordinates (NDC) → **viewport transform** → pixel coordinates.

A minimal model-view-projection in JavaScript (gl-matrix):

```js
import { mat4 } from "gl-matrix";
const proj = mat4.perspective(mat4.create(), Math.PI / 3, 16 / 9, 0.1, 1000);
const view = mat4.lookAt(mat4.create(), [0, 2, 5], [0, 0, 0], [0, 1, 0]);
const model = mat4.fromYRotation(mat4.create(), performance.now() / 1000);
const mvp = mat4.multiply(mat4.create(), proj, mat4.multiply(mat4.create(), view, model));
```

**Vectors used everywhere:** dot product (lighting angles, projections), cross product (normals), normalization, reflection `r = d − 2(d·n)n`.

## 3. The real-time rasterization pipeline

```mermaid
flowchart LR
  A[App / Engine<br/>CPU: culling, draw calls] --> B[Vertex / Mesh shader<br/>transform vertices]
  B --> C[Tessellation / Geometry<br/>optional]
  C --> D[Clipping + perspective divide<br/>+ viewport]
  D --> E[Rasterizer<br/>triangle → fragments]
  E --> F[Early depth test]
  F --> G[Fragment/Pixel shader<br/>materials, lighting, textures]
  G --> H[Output merger / ROP<br/>depth, stencil, blending]
  H --> I[Framebuffer → post-processing → swap chain → display]
```

### 3.1 Rasterization itself
For each triangle the rasterizer evaluates three **edge functions**:

```
E(x, y) = (x − x0)(y1 − y0) − (y − y0)(x1 − x0)
pixel inside ⇔ all three edge functions ≥ 0 (with top-left fill rule for ties)
```

The same values produce **barycentric coordinates** (λ0, λ1, λ2) used to interpolate vertex attributes (UV, normals, color) — with **perspective-correct** interpolation (interpolate `attr/w` and `1/w`, then divide).

A tiny software rasterizer in C++:

```cpp
struct V2 { float x, y; };
float edge(V2 a, V2 b, V2 p) { return (p.x - a.x) * (b.y - a.y) - (p.y - a.y) * (b.x - a.x); }

void drawTriangle(V2 v0, V2 v1, V2 v2, uint32_t color, uint32_t* fb, int W, int H) {
    int minX = std::max(0, (int)std::min({v0.x, v1.x, v2.x}));
    int maxX = std::min(W - 1, (int)std::max({v0.x, v1.x, v2.x}));
    int minY = std::max(0, (int)std::min({v0.y, v1.y, v2.y}));
    int maxY = std::min(H - 1, (int)std::max({v0.y, v1.y, v2.y}));
    float area = edge(v0, v1, v2);
    for (int y = minY; y <= maxY; ++y)
        for (int x = minX; x <= maxX; ++x) {
            V2 p{x + 0.5f, y + 0.5f};
            float w0 = edge(v1, v2, p), w1 = edge(v2, v0, p), w2 = edge(v0, v1, p);
            if (w0 >= 0 && w1 >= 0 && w2 >= 0) {           // inside (CW winding)
                // barycentrics: w0/area, w1/area, w2/area -> interpolate attributes
                fb[y * W + x] = color;
            }
        }
}
```

GPUs do exactly this, massively parallel, in fixed-function hardware, emitting pixels in **2×2 quads** (needed for texture derivative computation).

### 3.2 Visibility: the depth buffer
Each pixel stores the nearest depth so far. A new fragment is kept only if nearer (`LESS` test). Precision is non-linear in `z`; **reversed-Z** (near=1, far=0 with floating-point depth) greatly improves precision. **Hi-Z** and **early-Z** reject hidden fragments before shading.

### 3.3 Textures and sampling
- UV coordinates map image texels to surfaces.
- **Mipmaps**: pre-filtered half-resolution chain; the GPU picks level from screen-space derivatives to avoid aliasing/shimmer.
- Filtering: nearest, bilinear, trilinear, **anisotropic** (for surfaces at grazing angles).
- **Block compression** (BC1–BC7, ASTC) keeps textures compressed in VRAM and decodes in texture units.

## 4. Lighting and materials

**The rendering equation** (Kajiya, 1986) — what all lighting approximates:

```
Lo(x, ωo) = Le(x, ωo) + ∫Ω fr(x, ωi, ωo) · Li(x, ωi) · (n · ωi) dωi
```

Outgoing light = emitted + sum over all incoming directions of (BRDF × incoming light × cosine).

**Physically Based Rendering (PBR)** — industry standard metallic/roughness model:

- Diffuse: Lambert (`albedo / π`)
- Specular: **Cook-Torrance microfacet** BRDF = `D·F·G / (4 (n·l)(n·v))`
  - D: GGX/Trowbridge-Reitz normal distribution (roughness)
  - F: Fresnel (Schlick approximation)
  - G: Smith geometric shadowing
- Energy conservation, image-based lighting (IBL) from environment maps.

See code in [Shaders](shaders.md).

**Shadows:** shadow mapping (render depth from the light, compare), cascaded shadow maps for sun, or [ray-traced shadows](ray-tracing.md).
**Global illumination:** baked lightmaps, light probes, screen-space GI, voxel GI, Lumen (UE5: software/hardware RT hybrid), path tracing.

## 5. Rendering architectures

| Technique | Idea | Trade-off |
|---|---|---|
| **Forward** | Shade each object with all lights as drawn | Simple, MSAA-friendly; cost ∝ objects × lights |
| **Deferred** | Write a **G-buffer** (albedo, normal, roughness, depth), then light in screen space | Many lights cheap; heavy bandwidth, transparency hard |
| **Forward+ / clustered** | Bin lights into 3D screen tiles/clusters; forward shade with only relevant lights | Modern default |
| **Visibility buffer** | Store triangle ID per pixel, shade later | Used with dense geometry (Nanite) |
| **Tile-based (mobile HW)** | GPU renders per tile in on-chip memory | Bandwidth savings |

**Virtualized geometry** (UE5 Nanite): clusters of ~128 triangles in a hierarchy, GPU-driven culling, software rasterization of tiny triangles — billions of source triangles.

## 6. Anti-aliasing and post-processing

- **MSAA**: multiple coverage samples per pixel, shaded once.
- **FXAA/SMAA**: edge-detecting post filters.
- **TAA**: jitter the camera sub-pixel each frame and accumulate history reprojected with **motion vectors** — foundation of [DLSS](dlss.md), FSR, XeSS.
- Post chain: tone mapping (HDR → display, ACES/AgX), bloom, depth of field, motion blur, color grading, film grain.

## 7. Frame delivery

- **Double/triple buffering** with a **swap chain**; **V-Sync** waits for vertical blank to avoid tearing.
- **VRR** (G-Sync, FreeSync, HDMI VRR) lets the display wait for the GPU.
- Frame pacing & latency: CPU simulation → render submission → GPU → scan-out; NVIDIA Reflex / AMD Anti-Lag shrink the render queue.

## 8. Graphics APIs and platforms

| API | Platforms | Style |
|---|---|---|
| **Direct3D 12** | Windows, Xbox | Explicit, low-level; DXR ray tracing; HLSL shaders |
| **Vulkan** | Linux, Windows, Android | Explicit; SPIR-V shaders |
| **Metal** | Apple | Explicit-ish |
| **OpenGL / D3D11** | Legacy, still common | Driver manages state and memory |
| **WebGL 2 / WebGPU** | Browsers | JS APIs; WebGPU uses WGSL |

On Windows the display stack is **DWM** (compositor) + DXGI + WDDM; on Linux, **Wayland** compositors (Mutter, KWin) or X11 atop DRM/KMS and Mesa. Tools: RenderDoc (both), PIX (Windows), Nsight Graphics, Radeon GPU Profiler.

A complete minimal WebGPU triangle (JavaScript + WGSL):

```js
const adapter = await navigator.gpu.requestAdapter();
const device = await adapter.requestDevice();
const ctx = document.querySelector("canvas").getContext("webgpu");
const format = navigator.gpu.getPreferredCanvasFormat();
ctx.configure({ device, format });

const module = device.createShaderModule({ code: `
@vertex fn vs(@builtin(vertex_index) i: u32) -> @builtin(position) vec4f {
  var p = array(vec2f(0, .5), vec2f(-.5, -.5), vec2f(.5, -.5));
  return vec4f(p[i], 0, 1);
}
@fragment fn fs() -> @location(0) vec4f { return vec4f(1, .4, 0, 1); }` });

const pipeline = device.createRenderPipeline({
  layout: "auto",
  vertex: { module, entryPoint: "vs" },
  fragment: { module, entryPoint: "fs", targets: [{ format }] },
});
const enc = device.createCommandEncoder();
const pass = enc.beginRenderPass({ colorAttachments: [{
  view: ctx.getCurrentTexture().createView(), loadOp: "clear", storeOp: "store",
  clearValue: { r: 0.1, g: 0.1, b: 0.1, a: 1 } }] });
pass.setPipeline(pipeline); pass.draw(3); pass.end();
device.queue.submit([enc.finish()]);
```

## 9. Color science essentials

- Store textures in **sRGB**, light in **linear** space; convert at read/write.
- **HDR** rendering uses floating-point buffers (RGBA16F); displays: HDR10 (PQ curve, Rec.2020), scRGB on Windows.
- Gamma ≈ 2.2 — naïve averaging in sRGB space is wrong.

## Further reading
- Akenine-Möller et al., *Real-Time Rendering* (4th ed.)
- Pharr, Jakob, Humphreys, *Physically Based Rendering* (free online, pbr-book.org)
- LearnOpenGL.com; "WebGPU Fundamentals"; Scratchapixel.com
