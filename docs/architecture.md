# Architecture and implementation plan

## 1. Purpose

ConstructWASMCanvas is a reusable bridge for embedding a custom WebAssembly-powered Canvas surface inside an exported Construct 2 or Construct 3 project.

The intended HTML export contains two rendering surfaces:

- The Construct runtime's own canvas, used for the normal Construct scene.
- A separate Canvas owned by the embedded WASM application.

The custom Canvas can be composed as an **overlay** above Construct or an **underlay** below it. Construct remains responsible for the surrounding game flow and can decide when the custom surface is created, shown, hidden, resized, paused, or disposed.

This project is not a game engine and should not contain game-specific simulation or rendering logic. It provides the host/plugin integration that lets an external WASM application bring its own rendering surface into a Construct-exported page.

## 2. Goals

- Support Construct 2 and Construct 3 through their respective plugin SDKs.
- Load a WASM module and its JavaScript/browser bridge from an exported project.
- Create and manage a dedicated Canvas element.
- Support overlay and underlay composition.
- Expose a small Construct-facing API for initialization, commands, events, resource registration, sizing, visibility, and disposal.
- Allow external components to provide images or sprite resources to the WASM application.
- Keep the WASM application independent of Construct-specific types and undocumented runtime internals.
- Make the bridge generic enough for multiple WASM applications and Canvas use cases.

## 3. Non-goals

- Implementing a game engine, combat system, animation system, or game-specific asset format.
- Replacing the Construct renderer.
- Requiring the WASM module to use Rust. Rust is a motivating use case, not a restriction.
- Assuming Construct 2 and Construct 3 share a plugin binary or identical SDK behavior.
- Depending on private Construct texture objects or undocumented runtime internals.
- Promising support for every Construct export target before testing it.

## 4. High-level architecture

    Construct 2 / Construct 3 project
        |
        | Event Sheets: configure, command, react to events
        v
    ConstructWASMCanvas plugin adapter
        |
        | versioned host API
        v
    Browser host / WASM bridge
        |                         |
        | lifecycle + resources   | rendering
        v                         v
    WASM application          Dedicated Canvas
        |
        +-- application-defined simulation, state and draw logic

The plugin adapter owns Construct SDK integration. A shared browser-side host module should own generic concerns such as WASM loading, Canvas lifecycle, resource registration, event dispatch, and diagnostics. The WASM module owns its application-specific behavior.

A possible source layout, to be validated during implementation:

- `plugin/c2/`: Construct 2 SDK adapter.
- `plugin/c3/`: Construct 3 SDK adapter.
- `runtime/host/`: shared browser host and lifecycle.
- `runtime/bridge/`: versioned JavaScript/WASM communication contract.
- `examples/minimal-wasm-canvas/`: smallest working integration fixture.
- `docs/`: architecture, API, compatibility, and authoring guides.

This is a proposed layout, not an existing repository structure.

## 5. Construct plugin responsibilities

The plugin should expose generic capabilities rather than application-specific actions.

Expected responsibilities:

- Load a configured WASM module and its required JavaScript glue.
- Create, attach, size, show, hide, and dispose the custom Canvas.
- Select overlay or underlay composition.
- Synchronize the Canvas bounds with the Construct viewport/container.
- Forward generic commands and configuration to the WASM host API.
- Receive asynchronous ready, error, resource, and application events.
- Expose events to Construct as triggers and payloads/expressions where supported.
- Register external resources and report their readiness.
- Provide useful diagnostics when loading or initialization fails.

The plugin should not expose hard-coded actions such as `StartFight`, `SetFighterHealth`, or `LoadPokemon`. Those belong to a consuming application or its own adapter layer. The generic plugin should expose operations such as `Initialize`, `SendCommand`, `RegisterImage`, `SetLayerMode`, `ResizeSurface`, and `Dispose`.

Construct 2 and Construct 3 use different SDK generations. Share the browser host and protocol where possible, but keep version-specific plugin adapters thin and explicit.

## 6. Canvas composition

### Overlay

The custom Canvas is positioned above the Construct canvas. Transparent pixels allow the Construct scene to remain visible beneath it.

Potential uses:

- A WASM-rendered fighting game or minigame over the Construct scene.
- A custom visualizer or simulation.
- A specialized rendering layer that shares the page with Construct UI.

### Underlay

The custom Canvas is positioned below the Construct canvas. Construct can render foreground objects, HUD, menus, and interface elements above the WASM output.

Potential uses:

- A custom animated background.
- A specialized world or scene renderer with Construct-owned UI.
- A WASM visual layer combined with ordinary Construct objects.

### Composition requirements

Do not treat CSS z-index as the entire solution. The implementation must identify a stable attachment point and stacking context in each supported export/runtime.

The host should define and test:

- Canvas creation and DOM attachment.
- Overlay/underlay ordering and transparent clearing.
- CSS positioning, viewport alignment, scaling, and clipping.
- Logical resolution versus CSS size and device-pixel ratio.
- Resize, fullscreen, orientation, and browser zoom behavior.
- Pointer and keyboard routing, including whether the custom Canvas receives pointer events.
- Visibility, focus loss, pause, scene transitions, and disposal.
- Behavior when Construct resizes or recreates its own canvas/container.

The custom renderer must own its own Canvas context. It must not assume ownership of or draw into Construct's internal rendering context.

## 7. External image and sprite registration

A central requirement is allowing the Construct project or another external component to provide graphics to the WASM application.

### 7.1 Resource IDs, not Construct objects

The host API should register an image under a stable logical resource ID. The WASM application refers to that ID in its configuration or draw data.

The platform-neutral WASM application contract should not require Construct SDK objects, DOM nodes, `HTMLImageElement`, `ImageBitmap`, `CanvasImageSource`, or JavaScript object references.

A resource descriptor may include:

- Stable resource ID and optional content/version hash.
- Resource kind: image, sprite sheet/atlas, or another explicitly supported type.
- Pixel width and height.
- Optional pixel-density scale.
- Encoded format or MIME type where relevant.
- Atlas frame rectangles.
- Optional pivot/origin and trim metadata.
- Optional animation frame order and timing metadata.

Start with a minimal image descriptor. Add atlas and animation fields when the consuming application needs them.

### 7.2 Registration flow

1. The Construct project selects a resource, or supplies an explicit URL/byte payload through the plugin.
2. The plugin obtains a supported source representation using documented SDK capabilities where available.
3. The browser host validates the descriptor and decodes the image if needed.
4. The host stores the resulting browser image object in a private resource registry and assigns/records a stable resource ID.
5. The WASM application receives the resource ID and platform-neutral metadata.
6. The WASM renderer references that ID when producing draw operations.
7. The browser renderer resolves the ID to the registered image and draws it on the custom Canvas.
8. The host reports ready/failure events to Construct.

The exact method for reading Construct-managed image data must be verified for each SDK. Do not reach into undocumented private texture objects. If the SDK does not provide a supported way to obtain image bytes or a browser image source, the plugin should accept an explicit URL or byte payload instead.

### 7.3 Transfer once, draw many times

Image bytes should be transferred and decoded during registration, not resent every frame.

Per-frame rendering should use compact references and geometry, such as:

- Resource ID.
- Atlas frame index or source rectangle.
- Destination position and dimensions.
- Flip, rotation, opacity, and layer/order fields as supported.

Avoid base64-encoding entire images on every tick. Avoid repeated full-pixel-buffer copies across the JavaScript/WASM boundary. Optimize transfer only after profiling, while keeping resource ownership explicit.

### 7.4 Readiness, errors, and lifetime

Image decode is asynchronous. The API must define behavior for:

- Resource loading, ready, and failed states.
- Missing IDs and invalid descriptors.
- Unsupported image formats and decode failures.
- Invalid atlas rectangles or dimensions.
- Duplicate IDs and explicit replacement/versioning.
- Releasing a resource while the WASM application still references it.
- Disposing the WASM instance while decode is pending.
- Cross-origin URLs and CORS restrictions.

A resource ID must not silently begin referring to unrelated image content. Resource replacement should be explicit and observable.

## 8. Versioned host API

The JavaScript host and WASM module need a small, explicit, versioned contract. The exact transport is an implementation decision; the conceptual surface should cover:

### Host-to-module operations

- Initialize with API version and configuration.
- Register/update/release image resources.
- Send application-defined commands and input.
- Resize or change the rendering viewport.
- Pause/resume/reset where supported by the module.
- Request a state snapshot or diagnostic information.
- Dispose the module and release resources.

### Module-to-host events

- Ready / initialization failed.
- Resource ready / resource failed.
- Application-defined semantic events.
- Runtime error.
- Shutdown/disposed.

Use structured payloads with stable IDs and explicit fields. Do not rely on parsing arbitrary human-readable strings as a protocol.

The host must expose readiness explicitly. Commands received before initialization should either be queued under a documented policy or rejected with a clear error. They must not be silently lost.

Avoid chatty per-property polling across the JS/WASM boundary. Batch commands, resource registrations, and state/event payloads where practical.

## 9. Loading, packaging, and security

- Package the WASM binary, generated JS glue, and any required support files as explicit plugin dependencies or project files using each Construct SDK's supported mechanism.
- Do not assume generated WASM glue will survive Construct 2's JavaScript minification/Closure ADVANCED_OPTIMIZATIONS unchanged. Test the actual export pipeline and configure externals/renaming boundaries as required.
- Verify the minimum browser features needed by the chosen WASM toolchain and Canvas APIs.
- Treat module URLs and external asset URLs as untrusted inputs; surface fetch, MIME, CORS, and compilation errors.
- Avoid eval-based dynamic code generation unless a specific supported deployment requires it and its security implications are documented.
- Do not claim compatibility with native wrappers, mobile exports, or embedded webviews without testing them independently.

The first proof of concept should target a normal browser HTML export. Expand the support matrix only after real validation.

## 10. Diagnostics

Provide actionable diagnostics rather than a generic failed-to-load state.

At minimum, report:

- Plugin adapter/version and host API version.
- WASM fetch/compile/instantiate failures.
- Initialization timeout or rejected initialization.
- Canvas attachment and resize failures.
- Resource ID, source type, and decode error for asset failures.
- Unsupported API version or command.
- Disposal and lifecycle errors.

Diagnostics should be available to Construct events and, in development mode, the browser console. Avoid logging full asset payloads or sensitive URLs/tokens.

## 11. Example use case: StreetChavesRust and Petmals SyncFight

StreetChavesRust is one motivating consumer, not a dependency of this generic repository.

In that integration, a Construct project such as Petmals Pink&Dark could:

- Load the StreetChavesRust WASM module.
- Create its fight Canvas as an overlay or underlay.
- Register the sprite sets and stage graphics selected by the Construct project.
- Send fighter IDs, moveset IDs, stage ID, and match rules.
- Start and control the fight through generic commands.
- React to semantic events such as hit, combo update, round finish, and fight finish.

StreetChavesRust would own its fight simulation and draw its own fight presentation. Construct would continue to own the surrounding game flow, menus, HUD, and scene composition.

This example must not cause ConstructWASMCanvas itself to acquire fighting-game-specific APIs or dependencies.

## 12. Implementation stages

1. **Contract and compatibility research**
   - Verify current Construct 2 and Construct 3 SDK extension points.
   - Identify supported dependency packaging and runtime lifecycle hooks.
   - Verify documented asset-source access paths.
   - Record browser/export targets for the first proof of concept.

2. **Standalone browser host**
   - Build a minimal HTML harness independent of Construct.
   - Load a tiny test WASM module.
   - Create, resize, show, hide, and dispose a custom Canvas.
   - Prove overlay and underlay composition.

3. **Resource registry**
   - Register one permitted test image from encoded bytes or a supported browser image source.
   - Resolve its stable ID in the renderer.
   - Test readiness, decode failure, replacement, and disposal.
   - Verify that image data is not resent per frame.

4. **Versioned bridge**
   - Implement initialization, commands, events, resource registration, errors, and disposal.
   - Add API version negotiation and explicit readiness behavior.

5. **Construct 2 adapter**
   - Integrate the shared host with an actual Construct 2 exported HTML project.
   - Validate the generated JS/WASM packaging and minification path.
   - Validate overlay/underlay, resize, lifecycle, and asset registration.

6. **Construct 3 adapter**
   - Implement a separate adapter for the C3 SDK.
   - Reuse the host and protocol where possible.
   - Validate C3 packaging and runtime behavior independently.

7. **Examples and compatibility matrix**
   - Add minimal sample projects for supported targets.
   - Record browser/export compatibility and known limitations.

Do not skip the standalone harness. It isolates browser/WASM/Canvas problems from Construct SDK problems and makes the generic host contract testable before either plugin adapter is introduced.

## 13. Open decisions

- Exact C2 and C3 SDK APIs for plugin lifecycle, dependency packaging, and image-source access.
- Whether the first resource API accepts URLs, encoded bytes, SDK-provided image sources, or a restricted subset.
- Whether the browser host draws directly to Canvas or consumes renderer commands produced by the WASM module.
- Exact WASM binding/toolchain and generated-glue compatibility with the targeted Construct exports.
- Whether one plugin instance owns one WASM instance/Canvas or multiple independent instances.
- Whether commands before readiness are queued or rejected.
- Exact API version negotiation and event queue/backpressure behavior.
- Audio ownership: Construct, the embedded application, or a later optional bridge.

These are research and proof-of-concept decisions. Do not fill them with assumptions or claim SDK support before testing.

## 14. Acceptance criteria for the first working prototype

- A Construct-exported HTML page loads a small WASM module through the plugin.
- The plugin creates a separate Canvas without modifying Construct's rendering context.
- The Canvas can be displayed as both overlay and underlay.
- The Canvas tracks the Construct viewport through resize/fullscreen changes in the tested target.
- Construct can send a command and receive a structured event.
- Construct can register one external test image; the WASM application can reference it by stable resource ID and the Canvas renderer can draw it.
- Resource loading failure is observable by Construct.
- Disposing the plugin releases the Canvas, event handlers, module instance, and registered resources.
- The tested Construct version, export mode, browser, and limitations are documented.

A prototype that only draws a hard-coded rectangle is a useful first Canvas smoke test, but it does not satisfy the external asset-registration acceptance criterion.
