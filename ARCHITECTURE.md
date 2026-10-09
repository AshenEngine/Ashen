# Ashen Engine - Architecture & Design

## 1. Overview

Ashen is a pure-Rust 3D game engine designed around three core pillars: **performance, modularity, and developer ergonomics**.

The engine synthesises proven architectural patterns from modern game development ecosystems:

- **Bevvy** → Data-oriented ECS storage, explicit system scheduling, and a decoupled plugin architecture.
- **Hazel (Cherno)** → Thread-decoupled render command queues and UUID-backed asset handles.
- **Unity** → Unity-inspired visual editor workflows, inspector-driven composition, and intuitive prefab ergonomics.
- **Unreal** → Modern render-graph architecture and production-focused asset pipelines.

### The Hybrid Model

Ashen eliminates the friction between low-level performance and high-level usability by decoupling internal storage from external representation:

```rust
┌────────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER / EDITOR UX                           │
│     Entity-Component-Hierarchy Model · Prefabs · Visual Tooling        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Ergonomic Public API
┌───────────────────────────────────▼────────────────────────────────────┐
│                         ENGINE RUNTIME CORE                            │
│     Archetype-based ECS · Thread-Parallel Schedules · Render Queue     │
└────────────────────────────────────────────────────────────────────────┘

```

---

## 2. Design Goals

### Primary Goals

- **Data-Oriented Performance:** High performance through cache-friendly data layouts and parallel system execution.
- **Integrated Tooling:** A responsive visual editor co-developed alongside the runtime as a first-class citizen.
- **Extreme Modularity:** Decoupled subsystems (Physics, Audio, Input) that can be swapped or stripped out entirely.
- **Thread-Decoupled Rendering:** Game logic records rendering intentions into a command queue, isolating state execution from GPU submission.
- **Ergonomic Public API:** Approachable Rust abstractions that hide ECS boilerplate without sacrificing raw performance.
- **Rapid Asset Iteration:** Asynchronous asset streaming with live filesystem hot-reloading.

### Non-Goals (Early Phase)

* Immediate feature parity with monolithic commercial engines (Unity/Unreal).
* Universal platform targets (initial releases focus strictly on desktop platforms).
* Built-in visual scripting (planned for post-1.0 milestones).
* Sacrificing core architecture for short-term beginner abstractions.

---

## 3. High-Level Architecture & Data Flow

```rust
┌────────────────────────────────────────────────────────────────────────┐
│                              EDITOR LAYER                              │
│         Hierarchy · Component Inspector · Viewport · Gizmos            │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Shared World / IPC
┌───────────────────────────────────▼────────────────────────────────────┐
│                            PUBLIC API LAYER                            │
│           Entity · Components · Transform Hierarchy · Prefabs          │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼────────────────────────────────────┐
│                               CORE LAYER                               │
│                 App · World · Schedules · Plugin System                │
└──────────┬────────────────────────┬─────────────────────────┬──────────┘
           │ System Execution       │ Asset Handles           │ Draw Commands
┌──────────▼──────────┐  ┌──────────▼──────────┐  ┌───────────▼──────────┐
│  Physics & Logic    │  │   Asset Pipeline    │  │ Render Command Queue │
│  (Parallel Systems) │  │  (Async / Reload)   │  │   (Hazel Pattern)    │
└─────────────────────┘  └─────────────────────┘  └───────────┬──────────┘
                                                              │ Graph Execution
                                                  ┌───────────▼──────────┐
                                                  │ Compute-Driven Graph │
                                                  │  (Vulkan / DX12)     │
                                                  └───────────┬──────────┘
                                                              │ Shaders
                                                  ┌───────────▼──────────┐
                                                  │ Slang Pipeline (.spv)│
                                                  └──────────────────────┘

```

---

## 4. Core Subsystems

### 4.1 App & Plugin Architecture

The runtime lifecycle is orchestrated by a central `App` builder that owns system schedules, world state, and modular plugins. Engine features exist as isolated plugins, allowing developers to compose minimal, specialised runtimes by enabling only the modules required for their project.

### 4.2 ECS Core & Public API Bridge

Internally, components are packed into contiguous memory blocks (archetypes) for maximum cache locality during parallel system execution.

Externally, the public API abstracts query complexity into an intuitive entity-component model, offering a familiar object-composition workflow while maintaining data-oriented efficiency underneath.

### 4.3 Scene Graph & Transform Hierarchy

A relational transform propagation system sits on top of the ECS storage:

- Parent/child entity relationships.
- Hierarchical local-to-world matrix calculations.
- Deterministic scene serialisation using 128-bit UUID entity handles (inspired by Hazel).

### 4.4 Prefab & Override System

Prefabs exist as reusable asset templates that track property modifications:

- Nested prefabs with live template propagation.
- Field-level overrides that preserve parent linkages while customising individual instances.

### 4.5 Rendering Architecture

Built on top of native Vulkan and DirectX 12 to allow unrestricted access to modern hardware extensions.

- **Slang Shader Pipeline:** Instead of a restricted safe shading language, Ashen uses Slang as its universal shading frontend. Shaders leverage Slang’s generics and interfaces to scale cleanly between legacy and high-end hardware profiles. The engine's asset pipeline compiles Slang down to optimised SPIR-V for Vulkan and DXIL for DirectX 12, maximising GPU pipeline performance.
  * Slang is a modern, cross-platform shading language that compiles clean high-level abstractions into target-optimised GPU bytecode.
- **Render Command Queue (Hazel Pattern)**: Systems record rendering intentions into a thread-safe, double-buffered command queue during frame processing. The render thread consumes these commands independently of game logic.
- **Compute-Driven Framework:** Vertex culling, frustum testing, and occlusion checks are evaluated entirely on the GPU via Slang compute shaders. The CPU records wide draw boundaries, which are culled and executed directly on-hardware using indirect drawing (`vkCmdDrawIndexedIndirect` / `ExecuteIndirect`), keeping the CPU fully decoupled from geometric load.
- **Data-Driven FrameGraph:** The rendering lifecycle is managed by an explicit FrameGraph. Render passes (shadows, G-Buffer, clustered lighting, post-processing) declare their memory dependencies and resource inputs/outputs upfront. The graph automatically schedules async compute work, minimises GPU execution barriers, and dynamically strips out passes on lower-end hardware targets.

### 4.6 Asset Pipeline

- **UUID-Based Handles:** Resources are accessed via strongly-typed, immutable asset handle references rather than raw filesystem paths.
- **Asynchronous I/O:** Background thread pools handle disk reads and mesh/texture parsing without stalling the main execution loop.
- **Live Hot-Reloading:** File system watchers automatically re-upload modified shaders, textures, and material files at runtime.

### 4.7 Editor Architecture
The Ashen Editor is treated as a first-class part of the engine rather than an afterthought. 
It provides a 3D viewport, hierarchy, component inspector, and transform gizmos. 

The editor is designed to work closely with the runtime while keeping the final game binary clean of editor-only code.

---

## 5. Customizability & Usage Models

Ashen supports two primary integration paths:

1. **Full Turnkey Engine:** A ready-to-use distribution featuring the visual editor, default renderer, and standard toolsets.
2. **Modular Framework:** A lightweight foundation where developers import only the core ECS and rendering crates, assembling bespoke engines for specialised hardware or application constraints.

---

## 6. Architectural Boundaries

To maintain long-term code quality, Ashen strictly avoids:

- **Hidden Engine State:** Magic global singletons and untracked execution side-effects.
- **Monolithic Locking:** Tight coupling between logic updates and GPU draw execution.
- **Code-Only Workflows:** Forcing all scene creation and iteration to happen exclusively in code.

---

## 7. Active Design Decisions

The following areas are still under evaluation:

- **ECS Storage:** Custom archetype implementation vs existing libraries (`bevy_ecs`, `hecs`, etc.).
- **Editor UI Framework:** Immediate-mode vs retained-mode approaches.
- **Scripting Strategy:** Native Rust hot-reloading vs sandboxed embedded languages (planned after core systems stabilise).
- **Plugin Loading:** Dynamic plugin support is a longer-term goal and not required for initial versions, but would be great to have.

### Note
- This was AI-assisted, not fully generated; my ideas were just refined and made better and more readable. 
