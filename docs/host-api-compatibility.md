# WASM application host API: compatibility contract

## Status and intent

This document defines the proposed host-facing contract for ConstructWASMCanvas. It is documentation for a future implementation, not an implemented plugin, stable ABI, or compatibility guarantee.

The plugin must host arbitrary WASM applications, not only StreetChavesRust. A WASM application may be written in Rust, C/C++, Zig or another language. The plugin therefore integrates with a small JavaScript facade supplied by the application package rather than depending on one compiler's generated WASM ABI.

## Architecture boundary

The integration has three parts:

1. **Construct adapter:** version-specific Construct 2 or Construct 3 plugin code. It translates Construct actions/events into generic host operations.
2. **Shared browser host:** loads the application facade and WASM assets, owns the dedicated Canvas element, manages composition, resources, lifecycle, commands, events and diagnostics.
3. **Application facade:** a JavaScript entry point supplied by the WASM application package. It hides toolchain-specific WASM loading and connects the application to the host.

The application core must not depend on Construct SDK types or DOM objects. The plugin must not contain application-specific game logic. C2 and C3 adapters remain separate even if they reuse the same shared browser host.

## Application facade

Each application package exposes a facade with these semantic operations:

| Operation | Purpose |
|---|---|
| `mount(options)` | Attach to the host-created Canvas and receive host services |
| `initialize(config)` | Initialize and negotiate compatibility |
| `send(command)` | Deliver an application-defined command or normalized input |
| `registerResource(descriptor, source)` | Register an external resource once |
| `releaseResource(resourceId)` | Release a previously registered resource |
| `resize(viewport)` | Apply viewport and backing-store changes |
| `setPaused(paused)` | Pause or resume application progression |
| `dispose()` | Stop work and release application-owned resources |

These are semantic requirements, not a finalized JavaScript class signature. The host may settle on a concrete object shape during the first browser prototype. Asynchronous operations may return Promises and must report structured success/failure.

This facade allows a Rust application to use wasm-bindgen internally while keeping generated glue private. It also permits other WASM toolchains without changing the Construct plugin.

## Version negotiation

The host exposes its API version and capabilities before application initialization. The application declares the major version it supports.

- An incompatible major version fails initialization with a stable error code.
- Additive optional fields are backward-compatible.
- Unknown optional fields are ignored.
- Required-field or semantic changes require a major version increment.
- Capability checks use explicit feature identifiers, not Construct version-string heuristics.

The host must expose the negotiated version and capabilities in diagnostics and readiness events.

## Lifecycle and command delivery

Observable lifecycle states:

`created → loading → mounted → initializing → ready → disposed`

Failures during loading, mounting or initialization must produce a structured error. Paused/running is an orthogonal state after readiness.

Commands that require readiness must be rejected explicitly or queued under a documented bounded policy. Never silently drop commands received too early. Disposal must be idempotent and must cancel or safely finish pending loads/decodes without leaving event handlers, DOM nodes or timers behind.

## Canvas ownership and composition

The shared host creates and owns a dedicated Canvas element. It controls attachment, overlay/underlay placement, visibility, CSS geometry, resize observation and final DOM cleanup. The application facade mounts into the supplied Canvas and owns its application-specific rendering behavior.

The host must not draw into or take ownership of Construct's internal rendering context. Overlay and underlay are logical modes whose actual stacking implementation must be validated for each export/runtime. CSS z-index alone is not a compatibility guarantee.

The host communicates a viewport descriptor containing:

- logical width and height;
- CSS pixel width and height;
- backing-store pixel width and height;
- device-pixel ratio;
- optional viewport origin/offset;
- layer mode and visibility.

Logical coordinates must remain distinct from backing-store pixels. A resize must not implicitly reset application state. Pointer/keyboard routing and focus behavior are explicit host capabilities.

## Resource registration

The host exposes one-time registration of external images/resources. The plugin obtains a source through documented SDK facilities or an explicit URL/byte input. The shared host validates and decodes it, then retains the browser object in a private registry.

The application receives a stable resource ID and platform-neutral descriptor, never a Construct object, DOM node, ImageBitmap, HTMLImageElement or CanvasImageSource.

Minimal descriptor:

- stable resource ID;
- resource kind;
- pixel width and height;
- optional MIME type and content/version hash.

Atlas rectangles, pivots, trim metadata and animation descriptors are optional extensions. Registration reports asynchronous ready/failure events.

Define explicit behavior for duplicate IDs, replacement, release, unknown IDs, invalid dimensions, unsupported formats, CORS/decode failures and disposal during decode. Replacing a resource must be observable and must not silently change the meaning of an ID.

Do not inspect undocumented private Construct texture objects. If the SDK does not provide a documented image-source path, the plugin must accept an explicit URL or encoded bytes rather than reaching into runtime internals.

## Commands and events

The plugin transports generic structured commands to the application and structured events back to Construct. It does not interpret application-specific payloads.

Required generic event categories:

- host/application ready and initialization failure;
- resource ready and resource failure;
- runtime error;
- application-defined semantic event;
- disposed/shutdown.

Construct 2 and Construct 3 adapters map these to their own supported trigger/expression mechanisms. Avoid per-property polling every frame; prefer batched commands and events.

## Diagnostics

Every failure should include a stable error code, operation, application identifier/version, host API version and resource ID where relevant. Include actionable details for fetch, MIME, WASM compilation/instantiation, Canvas attachment, resize, decode, unsupported capability and disposal errors.

Development diagnostics may reach the browser console and Construct event system. Do not log binary payloads or sensitive URL query parameters.

## Rendering implementation freedom

The contract does not force one rendering implementation. The WASM application may render directly into the supplied Canvas through its browser adapter, or provide renderer-neutral draw data consumed by a browser-side renderer. Either is acceptable if Canvas ownership, resource identity, viewport semantics and lifecycle rules are preserved.

StreetChavesRust intends to implement its application side in Rust/WASM with Canvas 2D, alongside a separate native SDL3 backend. That is an example consumer, not a dependency or a required renderer.

## Minimum compatibility fixture

A first implementation is not complete until a small independent WASM sample can:

1. load through the shared host;
2. negotiate the API version;
3. mount and draw a visible test shape;
4. register and draw an external image by stable resource ID;
5. receive a host command and emit a structured event;
6. respond to resize and pause/resume;
7. dispose without leaked Canvas nodes, listeners or pending resources.

Test the fixture in a standalone browser harness before integrating Construct. Then test C2 and C3 independently and publish a compatibility matrix with exact Construct versions, export modes and browsers.

## Non-goals

- Making the plugin a game engine or renderer framework.
- Requiring Rust, wasm-bindgen or any specific WASM compiler.
- Importing StreetChavesRust combat or asset concepts into the generic API.
- Accessing undocumented Construct internals.
- Claiming support for every C2/C3 export target before real tests.
- Freezing a low-level binary ABI before the JavaScript facade and browser fixture are validated.
