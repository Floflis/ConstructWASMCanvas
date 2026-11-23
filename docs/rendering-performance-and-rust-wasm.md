# Rendering performance study: Construct 2/3 and Rust + SDL3 on WebAssembly

## Status and scope

This is a source-backed architecture study and implementation plan. It is not a benchmark report, a claim that Construct internals are fully reverse-engineered, or a compatibility guarantee for any export target.

The study answers four questions:

1. How Construct 2 and Construct 3 render their own content efficiently.
2. What their plugin SDKs allow third-party renderers to do.
3. How a Rust application that uses SDL3 can reach a browser Canvas through WebAssembly.
4. Which route best fits a generic WASM Canvas host without sacrificing the option of high-performance rendering.

The key distinction is that an HTML `<canvas>` is a surface, not a rendering API. A Canvas can have a 2D context, a WebGL context, or another supported context. Those are materially different rendering paths.

## Executive findings

- **Construct 2:** its runtime has a batched WebGL renderer and a Canvas 2D fallback. A world plugin is expected to implement both `draw(ctx)` and `drawGL(glw)` if it draws inside Construct's own scene. The C2 SDK also exposes redraw invalidation; it is wasteful to request a redraw when nothing visual changed.
- **Construct 3:** the runtime renderer is abstracted behind high-level interfaces and may use WebGL or WebGPU. The renderer tracks persistent state and can discard redundant state changes. SDK v2 also supports custom HTML content in HTML layers, including a separate Canvas between Construct layers. The runtime can run in a worker using OffscreenCanvas, while DOM work is handled on the main thread through a message bridge.
- **SDL3 + Emscripten:** SDL's 2D renderer uses GLES2 internally, which Emscripten maps to WebGL. It renders into an HTML Canvas, but this is **not Canvas 2D**. SDL3 can select the target Canvas by CSS selector. The browser main loop must yield to the event loop; vsync/requestAnimationFrame is the normal efficient choice.
- **Rust + `wasm32-unknown-emscripten`:** Rust can participate in Emscripten's C/C++/SDL and JavaScript ecosystem. This is the lower-rewrite path for an SDL-based Rust application, but the final artifact and toolchain integration must be proven for the chosen Rust SDL3 crate/build setup.
- **Rust + `wasm32-unknown-unknown` + `wasm-bindgen`/web-sys:** this is a different architecture. Keep the game core, replace SDL presentation/platform calls with a browser backend that targets Canvas 2D or WebGL. It is not an automatic translation of SDL calls into Canvas calls.
- **Recommended host design:** own a dedicated HTML Canvas element, but do not acquire its context before the application selects its renderer. Support an SDL3/Emscripten WebGL mode and a direct Rust/WebAssembly browser-renderer mode as distinct application backends. Do not force every WASM game into Canvas 2D.

## 1. What "Canvas rendering" actually means

The browser's `<canvas>` element is a drawing surface. Its first successful `getContext()` call selects the context family for that element. A Canvas created for WebGL cannot subsequently be treated as a Canvas 2D surface, and vice versa.

For this project, distinguish:

| Term | Meaning |
|---|---|
| Canvas element | DOM surface that can be positioned/composited with Construct |
| Canvas 2D | Browser 2D drawing API: paths, text, `drawImage`, transforms and compositing |
| WebGL | GPU API exposed through a Canvas context; Emscripten maps OpenGL ES/GLES operations to it |
| WebGPU | Newer GPU API; Construct 3's renderer abstraction can use it, but that does not make an SDL/WebGL context a WebGPU renderer |
| OffscreenCanvas | Canvas surface usable away from the DOM thread where supported; it changes ownership and transfer/lifecycle rules |

The plugin should therefore negotiate a **surface renderer mode**, not assume that every application means Canvas 2D when it says "Canvas".

A host-created Canvas should remain context-unclaimed until the application adapter initializes it. The host may set DOM geometry, CSS, stacking, accessibility metadata and lifecycle, but must not call `getContext("2d")` or `getContext("webgl")` pre-emptively.

## 2. Construct 2 rendering model

### 2.1 Runtime renderer choices

Construct 2 renders through an HTML Canvas. Its project can use:

- Canvas 2D, a compatibility fallback;
- WebGL, normally the faster path where available and hardware-accelerated.

The renderer expression can identify the active runtime renderer as `canvas2d` or `webgl`. A fallback is not merely a visual detail: plugins that draw inside the Construct scene must support the renderer mode selected by the project.

### 2.2 WebGL batching

Construct 2's runtime WebGL renderer traverses visible objects in back-to-front/Z order and builds a queue of rendering commands. It removes redundant state changes and combines consecutive compatible quads into a single draw batch.

The main batch state described by the renderer author includes:

- texture;
- opacity;
- blend mode;
- shader/effect;
- one or more quads.

A batch can draw many sprites when they share compatible state. A texture, blend mode, opacity or effect change can split a batch. Ordering remains important: the renderer cannot arbitrarily reorder transparent sprites without changing visual composition.

Construct 2 also culls objects that are off-screen before they reach the GPU. Its export pipeline can pack animation frames into spritesheets, reducing texture switches compared with preview, where frames may be separate textures.

Practical consequences:

- Keep compatible texture/blend/effect state adjacent where the visual order permits.
- Prefer atlases/spritesheets for related animation frames.
- Avoid unnecessary effects and large transparent regions.
- Do not sort transparent content in a way that breaks Z-order semantics merely to chase fewer batches.
- Measure exported builds, not only editor preview.

### 2.3 Canvas 2D fallback

Canvas 2D generally needs more browser API calls per object than the batched WebGL path. The Construct 2 renderer author explicitly notes the additional CPU-side calculations and JavaScript/browser transitions compared with the WebGL batch path.

Canvas 2D can still be hardware accelerated by the browser, but this is device/driver/browser dependent. A GPU blacklist or software fallback can change performance substantially.

### 2.4 C2 plugin drawing hooks

The C2 JavaScript SDK documents two rendering callbacks for world instances:

- `draw(ctx)`: Canvas 2D context;
- `drawGL(glw)`: Construct's GL wrapper when WebGL is enabled.

The SDK requires visually equivalent output from both paths. The plugin must use the wrapper's supported methods rather than assuming direct ownership of the low-level GL context.

When an action changes visual state, the plugin sets `runtime.redraw = true`. The runtime does not redraw the canvas automatically every tick; setting the flag when nothing visual changed causes wasteful redraws.

These callbacks are for drawing an object **inside Construct's renderer**. They are not a general-purpose API for creating a second independently owned Canvas.

### 2.5 C2 integration implication

For ConstructWASMCanvas, the least coupled route is a separate DOM Canvas surface owned by the host, positioned relative to the Construct runtime surface. The C2 adapter can use its plugin lifecycle/actions/events to control that surface.

If instead we choose to draw the WASM output as a C2 world instance, that is a different adapter design:

- Canvas 2D output must be drawn in `draw(ctx)`;
- WebGL output must be integrated through `drawGL(glw)`;
- renderer mode and fallback must be handled;
- the plugin must obey C2 redraw invalidation.

The two models should not be mixed implicitly. A DOM surface is the intended first architecture for a self-rendering SDL/WASM application.

C2's JavaScript is also processed by Closure Compiler during export. The host and WASM glue must be tested through the actual C2 export/minification pipeline; generated module glue, exported symbols and externals must not be assumed safe merely because they work in a standalone page.

## 3. Construct 3 rendering model

### 3.1 Renderer abstraction

Construct 3 exposes high-level renderer interfaces instead of requiring addons to manage raw GPU buffers. The runtime `IRenderer` interface is designed to work regardless of whether the underlying rendering technology is WebGL or WebGPU.

The interface has persistent state. Drawing code should explicitly establish the state it relies on:

- blend mode;
- fill mode;
- color/alpha;
- texture when using texture fill;
- other applicable 3D state.

The renderer can discard redundant state changes. Quads can then be submitted with `rect()` or `quad()` operations.

This is the same broad performance principle as C2's batching: reduce unnecessary state churn and submit compatible geometry efficiently. However, C3's abstraction must not be treated as a raw WebGL context, and a plugin should not assume it can inject an arbitrary foreign GPU context into Construct's renderer.

### 3.2 Addon SDK v2 and separate HTML surfaces

Construct 3 Addon SDK v2 provides access to HTML layers for inserting custom HTML content, including custom Canvas elements between Construct layers. This is directly relevant to the overlay/underlay design.

This gives the C3 adapter a documented direction for a separate Canvas surface:

- use the HTML-layer mechanism for DOM surface placement;
- let the host own the Canvas element and its CSS geometry;
- synchronize it with the relevant Construct layer/viewport;
- keep the WASM renderer's context independent from Construct's renderer.

C3's world-instance `Draw(renderer)` path is the alternative when the addon wants to draw through Construct's renderer abstraction. It is not the same as inserting an independent Canvas.

### 3.3 Worker mode and DOM boundary

Construct 3 can host its runtime in a dedicated Web Worker. In this mode, its renderer uses OffscreenCanvas. Addon runtime scripts then execute in the worker, while DOM scripts execute on the main thread.

The documented pattern is to split work:

- **Runtime side:** logic that can run in the worker; sends a message describing a DOM operation.
- **DOM side:** accesses `document`, creates/manages DOM elements, handles browser-only APIs, and replies asynchronously.
- **Bridge:** messaging connects the two sides. A call that would have been synchronous can become asynchronous.

SDK v2 recommends supporting worker mode where possible. It also allows addons to opt out, but that should be an explicit compatibility decision rather than the default escape hatch.

### 3.4 C3 integration implication

The C3 adapter should use the DOM-side addon APIs for Canvas element creation and HTML-layer placement. The runtime side should communicate through the SDK's messaging mechanism.

Do not assume that the WASM renderer can simply run inside the Construct worker. SDL's Emscripten guidance says rendering must be done on the browser main thread. This is an important constraint for the SDL3/Emscripten route.

For a first C3 implementation, keep the host-owned Canvas and SDL/Emscripten rendering on the main thread through the DOM side. The simulation/core may later be moved to a worker only after measuring and validating the chosen WASM toolchain, data exchange, and browser support.

A separate Rust browser renderer may support a worker/OffscreenCanvas architecture, but that must be validated independently. The existence of OffscreenCanvas in C3 does not automatically make every renderer or WASM runtime worker-compatible.

### 3.5 Context loss and resource recreation

C3's SDK provides renderer-context-loss hooks for addons that create renderer resources. If the addon uses Construct's renderer, it must drop invalid GPU resource references on loss and recreate them after restoration.

A separately owned WebGL Canvas has its own context-loss lifecycle. The host contract should surface context-lost/restored events to the application facade. Resource IDs should remain stable while GPU-side textures/buffers are recreated behind the registry.

## 4. Performance principles from Construct

Construct's official performance documentation identifies both CPU and GPU bottlenecks. The most relevant rendering lessons are:

### 4.1 Measure CPU and GPU separately

- CPU pressure often comes from events, JavaScript, behaviors, object creation and per-frame loops.
- GPU pressure comes from rendering work and effects.
- FPS is the final user-visible measure; CPU/GPU profilers help identify the bottleneck.
- Microbenchmarks should use controlled conditions and target devices.

### 4.2 Fill rate and overdraw

Large transparent images still consume fill rate. Overlapping images write the same pixels repeatedly. Layers rendered to their own texture require a later full-surface copy/composite. Effects can add extra passes and render-target work.

This applies to the separate WASM Canvas too. A transparent full-screen overlay is not free simply because most pixels are invisible. Prefer tight sprite bounds, limited overdraw and a surface resolution appropriate to the visual content.

### 4.3 Avoid readbacks

GPU-to-CPU pixel readback can stall the rendering pipeline. Construct's Drawing Canvas uses asynchronous snapshots specifically to avoid blocking the renderer while pixels are copied back.

The WASM bridge should avoid synchronous readback of a WebGL framebuffer or repeated full-canvas pixel extraction. If a screenshot or pixel inspection is required, make it an explicit asynchronous operation.

### 4.4 Avoid unnecessary frames

C2's redraw flag and C3's `runtime.sdk.updateRender()` express the same useful principle: request rendering when visible state changes, not on every no-op update.

For continuously animated games, use a display-synchronized loop. For static or paused surfaces, stop producing frames or render only when state changes. Keep simulation tick rate independent of display refresh rate.

## 5. Rust + SDL3 + WebAssembly: two distinct routes

### Route A: keep SDL3 through Emscripten

Rust supports the `wasm32-unknown-emscripten` target. It uses Emscripten's toolchain and can interoperate with its C/C++ ecosystem, SDL APIs, JavaScript glue and browser APIs.

SDL3's Emscripten backend provides a relatively direct porting route:

1. Compile the Rust application for `wasm32-unknown-emscripten`.
2. Link against an SDL3 build available to the Emscripten toolchain.
3. Keep using SDL's window, input, event and renderer abstractions where supported.
4. SDL's 2D renderer uses GLES2 internally; Emscripten maps those operations to WebGL.
5. SDL creates/uses an HTML Canvas element for its window.
6. Configure the target element with `SDL_HINT_EMSCRIPTEN_CANVAS_SELECTOR` before creating the SDL window.
7. Run the main loop through SDL callbacks or an Emscripten loop that yields to the browser.
8. Enable vsync (SDL renderer vsync or GL swap interval 1) so presentation can use `requestAnimationFrame`.

**Important:** this produces a WebGL-backed Canvas, not a Canvas 2D context. It is still an HTML Canvas surface and can be composed as a separate DOM layer.

Benefits:

- Preserves much of the existing SDL-facing application code.
- Uses a GPU-backed WebGL path rather than copying a software framebuffer to Canvas 2D every frame.
- SDL handles substantial platform integration: window, input, events, renderer and audio abstractions.
- SDL3 provides a Canvas selector hint, which is useful for targeting the host-created surface.

Costs and constraints:

- The Rust target is Tier 2 and requires the Emscripten toolchain.
- The Rust SDL3 crate/build integration must be validated; the existence of the Rust target does not prove a particular crate version or build script works.
- Native dependencies/middleware must have Emscripten-compatible builds.
- The browser main loop must return to the event loop; an infinite native-style loop is not suitable.
- SDL's Emscripten guidance says rendering must happen on the main thread.
- WebAssembly threads require a separate threaded build and appropriate COOP/COEP headers; do not make them a prerequisite for the first compatibility target.
- Browser audio may require a user gesture before playback is audible.
- The actual SDL3/Emscripten output must be tested inside Construct's export packaging and lifecycle.

This route is best when minimizing changes to an existing SDL3 renderer is more important than using Canvas 2D specifically.

### Route B: replace the SDL platform/presentation adapter with a browser Canvas renderer

Use `wasm32-unknown-unknown` with `wasm-bindgen` and `web-sys` (or another WASM/browser binding layer).

The application is split into:

- platform-neutral simulation and game logic;
- renderer-neutral presentation data or draw commands;
- native SDL3 adapter;
- browser adapter that owns a Canvas 2D or WebGL context.

In this route, SDL is not compiled to the browser and SDL calls are not automatically translated. The browser adapter implements the same application-facing interface that the SDL adapter implements natively.

For a Canvas 2D backend:

- Rust can call browser APIs through generated bindings.
- Use `requestAnimationFrame` for the render cadence.
- Use a fixed-step simulation accumulator independent of render cadence.
- Batch draw data in Rust and cross the WASM/JS boundary in coarse chunks, not one host call per sprite when avoidable.
- Cache decoded browser images and atlas metadata by stable resource ID.
- Keep Canvas state changes predictable; avoid repeatedly setting unchanged transforms, alpha, composite mode and smoothing flags.
- Avoid `getImageData()`/synchronous pixel readbacks in the hot path.
- Scale the backing store intentionally using CSS dimensions and device-pixel ratio; cap render resolution when fill rate becomes the bottleneck.

For a WebGL backend:

- The browser adapter can issue WebGL commands directly, or use a Rust rendering library with a web backend.
- Batch geometry and texture bindings.
- Manage context loss/restoration.
- Avoid mixing ownership of the context with Construct's renderer.

Benefits:

- Clean separation of game logic from native SDL and browser APIs.
- A true Canvas 2D option if that is a requirement.
- More direct integration with a host facade and resource registry.
- Better ability to choose web-specific scheduling and rendering behavior.

Costs:

- Requires implementing and maintaining a second renderer/platform adapter.
- A naive per-draw `web-sys` call pattern can create excessive WASM-to-JavaScript crossings.
- Canvas 2D may not match WebGL performance for large sprite counts/effects.
- Browser-specific resource, input, audio, resize and lifecycle code must be implemented.
- The native SDL renderer and browser renderer can drift unless they consume the same presentation contract and share conformance tests.

This route is best when the project wants a true browser-native renderer and a platform-neutral core, rather than a minimal SDL port.

### Route C: software framebuffer upload

A third possibility is to render into a CPU-side RGBA framebuffer and upload pixels to Canvas 2D using `ImageData`/`putImageData`, or upload the buffer as a texture to WebGL.

This is useful as a diagnostic, compatibility fallback or first smoke test. It is not the recommended primary high-performance renderer:

- Every frame can require a full framebuffer transfer.
- Pixel copies and format conversion consume CPU and memory bandwidth.
- It bypasses much of the benefit of GPU-side sprite batching and texture residency.
- At larger resolutions, the upload cost scales with the number of pixels.

If used, make it an explicit fallback backend and benchmark it at the target logical resolution. Do not confuse this with SDL3's Emscripten WebGL renderer.

## 6. What "SDL -> Canvas" should mean for StreetChavesRust

The term is ambiguous. There are two valid meanings:

1. **SDL3 continues to exist in the WASM build:** SDL3/Emscripten creates a WebGL-backed Canvas. The application still uses SDL APIs. This is the lowest-friction migration route.
2. **SDL3 is replaced at the browser boundary:** the game core remains Rust, but the browser build uses a Rust/WebAssembly Canvas renderer instead of SDL. This is the cleanest long-term cross-platform architecture.

Do not implement a third, hidden interpretation in which every SDL draw call is intercepted and converted to Canvas 2D calls. That would add an expensive and fragile compatibility layer, duplicate renderer semantics, and still not make SDL's native renderer equivalent to Canvas 2D.

### Recommended decision

Keep both routes possible in the architecture, but choose one primary proof of concept:

- First, investigate whether the current Rust SDL3 dependencies can build cleanly for `wasm32-unknown-emscripten`. This is a feasibility spike, not a commitment to keep SDL as the web renderer.
- In parallel, preserve the renderer-neutral core boundary and define the browser renderer contract.
- Compare a minimal SDL3/Emscripten WebGL sample with a minimal Rust/web-sys Canvas 2D sample using the same scene and asset set.
- Select the default only after measuring frame time, CPU time, GPU time, memory, startup cost, asset decode/upload, and input latency on target devices.

For StreetChavesRust, the engine must not depend on ConstructWASMCanvas. The plugin is one possible host for the browser artifact; a standalone HTML harness remains the first validation target.

## 7. Proposed host surface and renderer negotiation

The host should create a dedicated Canvas element but leave its context unclaimed. The application package declares a renderer requirement/capability set before mounting.

Conceptual capabilities:

- `canvas2d`
- `webgl1`
- `webgl2`
- optional `offscreen-canvas` where validated
- optional threading capability, only when the deployment headers and target support it

The host must not silently downgrade a required renderer. It should return a structured capability-negotiation error if the application requires WebGL2 but only Canvas2D is available.

The application facade should report:

- selected renderer backend;
- required and optional browser capabilities;
- host API version;
- canvas/context initialization result;
- context-lost/restored events;
- viewport and device-pixel ratio;
- lifecycle and disposal status.

The host owns the DOM element, stacking, CSS geometry and visibility. The application adapter owns the context and rendering resources after successful context acquisition. This split avoids Construct-specific rendering objects leaking into the WASM application.

### Avoid sharing Construct's renderer

The generic host must not attempt to acquire or reuse Construct's internal WebGL/WebGPU context. The renderer abstraction is an internal Construct service with its own state and lifecycle. C2 exposes a GL wrapper to C2 plugin drawing callbacks; C3 exposes high-level renderer interfaces. Neither is a promise that arbitrary external WASM code can take over the engine's GPU context.

Use a separate Canvas for the first implementation. If a future integration wants to draw through Construct's own renderer, implement that as a separate, SDK-specific presentation adapter.

## 8. Timing, threading and frame pacing

### One owner for presentation

Do not let both Construct and the WASM application independently drive the same Canvas. The embedded application owns its render loop; the host owns start/pause/resume/visibility/disposal signals.

Use `requestAnimationFrame` or SDL/Emscripten's browser-compatible main loop for display-synchronized presentation. The simulation should use a fixed logical tick and an accumulator so that display refresh changes do not change game rules.

### Avoid per-frame bridge chatter

A costly design is:

- Construct event calls a WASM function for each property;
- WASM calls JavaScript for each sprite;
- the host resolves each image independently;
- each sprite produces its own browser message.

Prefer:

- batched commands;
- typed arrays or compact binary draw packets where appropriate;
- stable resource IDs resolved from a persistent browser-side registry;
- one frame submission or one bounded set of submissions;
- semantic events sent only when they occur.

The exact transport must be measured. SharedArrayBuffer is not a default assumption because it requires cross-origin isolation and a threaded build.

### C3 worker mode

C3's runtime worker and SDL's Emscripten main-thread rendering requirement create a boundary that must be respected. The first SDL/Emscripten proof should keep rendering and the SDL window/context on the main thread through the DOM-side host integration. Do not claim worker compatibility until a real exported C3 project proves it.

## 9. Performance test plan

No performance numbers are asserted by this study. The following tests are required before choosing a default renderer.

### Test fixtures

Use one deterministic scene and the same assets across these variants:

1. Construct 2 native scene with its normal WebGL renderer.
2. Construct 2 native scene with Canvas 2D fallback where available.
3. Construct 3 native scene with its selected renderer (record WebGL/WebGPU).
4. Separate Canvas2D host surface with a minimal Rust/WASM browser renderer.
5. Separate WebGL Canvas with Rust + SDL3/Emscripten.
6. Optional CPU framebuffer upload baseline.

### Workloads

- Static frame (tests unnecessary redraw).
- 100, 500, 1,000 and 5,000 quads.
- Same texture vs alternating textures.
- Atlas animation vs separate textures.
- Opaque sprites vs large transparent borders.
- Low and high overdraw.
- Alpha/blend changes.
- Resize and device-pixel-ratio changes.
- Resource registration and cold/warm image decode.
- Pause/resume and context loss/restoration.
- Input-to-render latency.

### Measurements

- median and p95 frame time;
- FPS under unlimited/full-frame mode and normal display-synchronized mode;
- CPU/main-thread time;
- GPU time where available;
- JS/WASM boundary call count and bytes per frame;
- draw/batch count where instrumentable;
- texture count and GPU memory estimates where available;
- WASM startup/compile time;
- asset decode/upload time;
- memory growth and resource release after disposal.

Run on representative desktop and mobile hardware, and record browser, OS, GPU, Construct release, export mode, renderer and power mode. Use warm-up runs and repeated measurements. Never compare editor preview against an exported build as if they were the same pipeline.

### Acceptance gates

- No synchronous GPU readback in the hot path.
- No per-sprite host messaging.
- No unbounded command/event queues.
- No full-resolution transparent overlay unless required.
- No context acquisition before renderer negotiation.
- No main-thread infinite loop.
- No worker/thread claim without export-level validation.
- No renderer default selected solely from theoretical API speed.

## 10. Compatibility and risk matrix

| Area | Construct 2 | Construct 3 |
|---|---|---|
| Own renderer | Canvas2D or WebGL | Renderer abstraction; WebGL/WebGPU |
| Plugin drawing | `draw(ctx)` and `drawGL(glw)` | SDK renderer interfaces for plugin drawing |
| Separate DOM Canvas | Custom DOM integration must be validated against C2 exported page/runtime | SDK v2 HTML layers explicitly support custom HTML/Canvas content |
| Worker runtime | Not the C3 worker architecture | Runtime may run in worker with OffscreenCanvas |
| DOM ownership | Main-thread page integration | DOM-side script + runtime messaging when worker mode is active |
| Renderer sharing | Do not assume arbitrary access to internal GL context | Do not assume arbitrary access to internal WebGL/WebGPU context |
| Export constraints | Closure/minification and legacy SDK behavior must be tested | SDK v2 required for current addon architecture; DOM and worker behavior must be tested |

Additional risks:

- Construct 2 is legacy and has a different SDK/runtime model. Do not imply that a C3 SDK v2 addon can be reused in C2.
- Construct 3 SDK v2 is the current addon architecture; SDK v1 support was removed in r450+ according to the migration guide.
- C3's runtime worker mode does not automatically make an Emscripten SDL renderer worker-safe.
- A Canvas context type is selected once. Context ownership must be explicit.
- Browser hardware acceleration and GPU availability vary.
- Mobile memory and fill rate can become the bottleneck before CPU rendering code does.
- External assets may be subject to CORS and browser decode restrictions.
- Cross-origin isolation requirements make threaded WASM a separate deployment profile.

## 11. Implementation sequence

1. **Browser-only renderer spike:** create a standalone HTML host with one empty Canvas, no Construct and no game engine.
2. **SDL3/Emscripten spike:** compile a tiny Rust + SDL3 application; select the host Canvas; draw a few sprites; verify WebGL context, input, resize, pause and disposal.
3. **Rust Canvas2D spike:** use `wasm32-unknown-unknown` and web bindings; render the same fixture without SDL.
4. **Compare:** run the same scene/asset tests and record results.
5. **Finalize host renderer negotiation:** document required capabilities and context ownership.
6. **C2 adapter:** test the chosen host against a real exported C2 HTML project, including minification and overlay/underlay.
7. **C3 adapter:** use SDK v2 HTML layers and DOM-side scripts; test worker mode and DOM messaging independently.
8. **Consumer integration:** add StreetChavesRust only after the generic fixture passes.

## 12. Source references

### Construct 2

- [Technology overview: Canvas2D and WebGL](https://www.construct.net/en/construct-2/manuals/construct-2/overview/technology)
- [How the Construct 2 WebGL renderer works](https://www.construct.net/en/blogs/ashleys-blog-2/construct-webgl-renderer-works-917)
- [Construct 2 runtime functions: draw/drawGL and redraw](https://www.construct.net/en/construct-2/manuals/construct-2-javascript-sdk/sdk-documentation/runtime-functions)
- [Construct 2 performance tips](https://www.construct.net/en/construct-2/manuals/construct-2/overview/performance-tips)

### Construct 3

- [IRenderer script interface](https://www.construct.net/en/make-games/manuals/construct-3/scripting/scripting-reference/graphics-interfaces/irenderer-interface)
- [IWebGLRenderer interface](https://www.construct.net/en/make-games/manuals/addon-sdk/reference/graphics-interfaces/iwebglrenderer)
- [Addon SDK runtime scripts and worker/DOM-side split](https://www.construct.net/en/make-games/manuals/addon-sdk/guide/runtime-scripts)
- [Addon SDK v2 features, including HTML layers](https://www.construct.net/en/forum/construct-3/plugin-sdk-10/addon-sdk-v2-182122)
- [Porting to Addon SDK v2](https://www.construct.net/en/make-games/manuals/addon-sdk/guide/porting-addon-sdk-v2)
- [Construct 3 performance tips](https://www.construct.net/en/make-games/manuals/construct-3/tips-and-guides/performance-tips)
- [GPU Profiler](https://www.construct.net/en/make-games/manuals/construct-3/interface/debugger/gpu-profile-tab)

### Rust, SDL3 and browser APIs

- [Rust `wasm32-unknown-emscripten` target](https://doc.rust-lang.org/rustc/platform-support/wasm32-unknown-emscripten.html)
- [SDL3 Emscripten guide](https://wiki.libsdl.org/SDL3/README-emscripten)
- [SDL3 Canvas selector hint](https://wiki.libsdl.org/SDL3/SDL_HINT_EMSCRIPTEN_CANVAS_SELECTOR)
- [SDL3 Emscripten port in Emscripten source](https://github.com/emscripten-core/emscripten/blob/main/tools/ports/sdl3.py)
- [OffscreenCanvas API](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)

## 13. Open research questions

- Which Rust SDL3 crate version/build script can currently link against the selected Emscripten SDL3 port without local patches?
- Can the chosen SDL3/Emscripten artifact initialize on a Canvas element inserted through the Construct 3 SDK v2 HTML-layer/DOM-side mechanism?
- Does the target C2 export/minification pipeline preserve the generated Emscripten loader and module imports?
- What are the actual performance and memory differences between SDL3/Emscripten WebGL and Rust/web-sys Canvas2D for the same StreetChaves scene?
- Which asset-source mechanisms are officially exposed by the C2 and C3 SDKs for one-time external resource registration?
- What is the correct main-thread/worker arrangement for each supported export target?

These questions remain explicit implementation gates. The documentation does not pretend that source research substitutes for a real exported-project prototype.
