# ConstructWASMCanvas

A proposed integration layer for **Construct 2 and Construct 3** projects that need to embed custom WebAssembly-powered rendering surfaces, especially Canvas-based renderers, inside their exported HTML games.

The goal is to let a Construct project load a WASM module, create and manage its own Canvas surface, exchange commands and events with the module, and compose that surface above or below Construct's own renderer.

This repository is at its initial documentation stage. The plugin, runtime bridge, and SDK adapters are not implemented yet.

## Project goals

- Provide a reusable WASM + custom Canvas integration for Construct 2 and Construct 3.
- Keep Construct-specific SDK code at the host/plugin boundary.
- Support overlay and underlay composition with explicit sizing and lifecycle management.
- Define a small, versioned host-to-WASM contract for initialization, commands, events, resources, and disposal.
- Support external image/sprite registration without depending on undocumented Construct internals.
- Keep the integration generic so different WASM applications can use it, not just one game engine.

## Architecture

See [Architecture and implementation plan](docs/architecture.md) for the proposed boundaries, Canvas composition model, external asset registration, host API, compatibility research, implementation stages, and prototype acceptance criteria.

## Scope and compatibility

Construct 2 is a legacy product and Construct 3 has a different SDK/runtime model. Compatibility must be validated against real exported projects and target browsers. Do not assume one plugin binary or one asset-access mechanism will work unchanged across both products or every export target.

## Status

**Proposal / planning.** No runtime implementation or compatibility guarantees are currently provided.
