# Ultimaker Cura → Rust Port: Comprehensive Planning Document

## Executive Summary

This document outlines a strategy for porting Ultimaker Cura from Python/PyQt to Rust, with an API-first architecture enabling headless operation and multiple frontend options (MCP, web, desktop via Tauri).

### Key Design Principles

1. **API-First Architecture**: Core slicer logic exposed via a well-defined API
2. **Programmatic Porting**: Automated transpilation with manual refinement
3. **Upstream Tracking**: Re-runnable tooling for upstream version updates
4. **Modular Frontend**: UI as a separate project using Tauri + Bun
5. **Incremental Migration**: Phase-by-phase approach with hybrid operation

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Programmatic Porting Strategy](#2-programmatic-porting-strategy)
3. [Core Module Mapping](#3-core-module-mapping)
4. [Rust Crate Dependencies](#4-rust-crate-dependencies)
5. [API Design](#5-api-design)
6. [Frontend Architecture](#6-frontend-architecture)
7. [Implementation Phases](#7-implementation-phases)
8. [Tooling Specification](#8-tooling-specification)

---

## 1. Architecture Overview

### Current Python Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Ultimaker Cura (Python)                   │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   PyQt6 UI  │  │  QML Views  │  │  Signal System      │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
│         │                │                     │            │
│  ┌──────▼────────────────▼─────────────────────▼──────────┐ │
│  │              CuraApplication (Monolith)                 │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │ │
│  │  │ Settings │ │  Scene   │ │ Machines │ │  Plugins │  │ │
│  │  │  System  │ │  Graph   │ │  Manager │ │  System  │  │ │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘  │ │
│  └───────┼────────────┼────────────┼────────────┼─────────┘ │
│          │            │            │            │           │
│  ┌───────▼────────────▼────────────▼────────────▼─────────┐ │
│  │                 Uranium Framework                       │ │
│  └─────────────────────────┬───────────────────────────────┘ │
│                            │ Protobuf/Arcus                  │
│  ┌─────────────────────────▼───────────────────────────────┐ │
│  │              CuraEngine (C++ Backend)                    │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Target Rust Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         FRONTENDS (Separate Projects)                    │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Tauri App  │  │   Web UI    │  │  MCP Server │  │    CLI      │    │
│  │  (Desktop)  │  │  (Browser)  │  │  (AI/LLM)   │  │  (Scripts)  │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         │                │                │                │            │
│         └────────────────┼────────────────┼────────────────┘            │
│                          │                │                              │
│  ┌───────────────────────▼────────────────▼────────────────────────────┐│
│  │                         API LAYER (REST/gRPC/WebSocket)              ││
│  │  • Jobs: slice, arrange, export                                      ││
│  │  • Resources: machines, materials, profiles, scene                   ││
│  │  • Events: progress, errors, state changes (SSE/WS)                  ││
│  └──────────────────────────────┬───────────────────────────────────────┘│
└─────────────────────────────────┼───────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼───────────────────────────────────────┐
│                         cura-core (Rust Library)                         │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────────┐ │
│  │  cura-settings │  │   cura-scene   │  │      cura-machines         │ │
│  │  ───────────── │  │   ──────────── │  │      ──────────────        │ │
│  │  ContainerStack│  │  SceneNode     │  │  MachineDefinition         │ │
│  │  SettingDef    │  │  Decorators    │  │  MaterialProfile           │ │
│  │  Inheritance   │  │  BuildPlate    │  │  QualityProfile            │ │
│  │  Validation    │  │  Transforms    │  │  ContainerTree             │ │
│  └────────┬───────┘  └────────┬───────┘  └─────────────┬──────────────┘ │
│           │                   │                        │                 │
│  ┌────────▼───────────────────▼────────────────────────▼───────────────┐│
│  │                       cura-engine-bridge                             ││
│  │  ────────────────────────────────────────────────────────────────── ││
│  │  • Protobuf message generation (tonic/prost)                        ││
│  │  • Slicing job orchestration                                        ││
│  │  • G-code post-processing                                           ││
│  │  • Engine process management                                        ││
│  └──────────────────────────────┬───────────────────────────────────────┘│
│                                 │                                        │
│  ┌──────────────────────────────▼───────────────────────────────────────┐│
│  │  cura-formats                                                        ││
│  │  ────────────                                                        ││
│  │  • 3MF reader/writer (zip + XML)                                     ││
│  │  • G-code reader/writer                                              ││
│  │  • STL/OBJ/AMF mesh import                                           ││
│  │  • Profile serialization (.def.json, .inst.cfg)                      ││
│  └──────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐│
│  │  cura-devices                                                        ││
│  │  ────────────                                                        ││
│  │  • Printer discovery (mDNS/USB)                                      ││
│  │  • Output devices (network, USB, file)                               ││
│  │  • Device communication protocols                                    ││
│  └──────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    CuraEngine (C++ - Unchanged)                          │
│  ────────────────────────────────────────────────────────────────────── │
│  Communicates via Protobuf over TCP socket (existing protocol)          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Programmatic Porting Strategy

### 2.1 Why Programmatic Porting?

Manual porting of ~16,000 lines of Python is:
- Time-consuming and error-prone
- Difficult to keep synchronized with upstream
- Requires deep understanding of every code path

A programmatic approach provides:
- **Repeatability**: Re-run on each upstream release
- **Consistency**: Same transformation rules applied everywhere
- **Diffability**: Track what changed between versions
- **Incremental refinement**: Improve translator, not manual ports

### 2.2 Transpilation Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    py2cura-rs Transpilation Pipeline                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐│
│  │   Python     │     │     AST      │     │    Intermediate      ││
│  │   Source     │────▶│   Parser     │────▶│   Representation     ││
│  │   Files      │     │  (RustPython │     │   (Custom IR)        ││
│  │              │     │   parser)    │     │                      ││
│  └──────────────┘     └──────────────┘     └──────────┬───────────┘│
│                                                       │             │
│                                                       ▼             │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Transformation Passes                      │  │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐ │  │
│  │  │   Type      │ │  Pattern    │ │    Cura-Specific        │ │  │
│  │  │  Inference  │ │  Matching   │ │    Transforms           │ │  │
│  │  │  Pass       │ │  Pass       │ │    (Qt→channels, etc)   │ │  │
│  │  └─────────────┘ └─────────────┘ └─────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                       │             │
│                                                       ▼             │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐│
│  │    Rust      │     │    Code      │     │    Post-Generation   ││
│  │    Source    │◀────│   Generator  │◀────│    Fixups            ││
│  │    Files     │     │  (syn/quote) │     │    (cargo fmt, etc)  ││
│  │              │     │              │     │                      ││
│  └──────────────┘     └──────────────┘     └──────────────────────┘│
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 Transpilation Rules

The transpiler applies domain-specific transformations:

#### Python → Rust Type Mappings

| Python Type | Rust Type |
|-------------|-----------|
| `str` | `String` |
| `int` | `i64` |
| `float` | `f64` |
| `bool` | `bool` |
| `List[T]` | `Vec<T>` |
| `Dict[K, V]` | `HashMap<K, V>` |
| `Optional[T]` | `Option<T>` |
| `Set[T]` | `HashSet<T>` |
| `Tuple[A, B]` | `(A, B)` |
| `Any` | `Box<dyn Any>` or generic `T` |
| `Union[A, B]` | `enum` with variants |
| `Callable[[Args], Ret]` | `Fn(Args) -> Ret` |

#### Qt Signal → Rust Channel Mapping

```python
# Python (Qt Signal)
class MachineManager(QObject):
    globalContainerChanged = pyqtSignal()

    def setActiveMachine(self, stack):
        self._global_container_stack = stack
        self.globalContainerChanged.emit()
```

```rust
// Rust (tokio broadcast channel)
pub struct MachineManager {
    global_container_changed: broadcast::Sender<()>,
    global_container_stack: Option<Arc<GlobalStack>>,
}

impl MachineManager {
    pub fn set_active_machine(&mut self, stack: GlobalStack) {
        self.global_container_stack = Some(Arc::new(stack));
        let _ = self.global_container_changed.send(());
    }

    pub fn subscribe_global_container_changed(&self) -> broadcast::Receiver<()> {
        self.global_container_changed.subscribe()
    }
}
```

#### Singleton → Lazy Static / OnceCell

```python
# Python Singleton
class ContainerRegistry:
    __instance = None

    @classmethod
    def getInstance(cls):
        if cls.__instance is None:
            cls.__instance = ContainerRegistry()
        return cls.__instance
```

```rust
// Rust with once_cell
use once_cell::sync::Lazy;
use std::sync::Arc;
use parking_lot::RwLock;

static CONTAINER_REGISTRY: Lazy<Arc<RwLock<ContainerRegistry>>> =
    Lazy::new(|| Arc::new(RwLock::new(ContainerRegistry::new())));

impl ContainerRegistry {
    pub fn instance() -> Arc<RwLock<ContainerRegistry>> {
        Arc::clone(&CONTAINER_REGISTRY)
    }
}
```

#### Decorator Pattern → Trait Objects

```python
# Python Decorator Pattern
class CuraSceneNode(SceneNode):
    def callDecoration(self, name, *args):
        for decorator in self._decorators:
            if hasattr(decorator, name):
                return getattr(decorator, name)(*args)
```

```rust
// Rust with trait objects
pub trait SceneDecorator: Send + Sync {
    fn name(&self) -> &'static str;
    fn as_any(&self) -> &dyn Any;
}

pub struct CuraSceneNode {
    decorators: Vec<Box<dyn SceneDecorator>>,
}

impl CuraSceneNode {
    pub fn get_decorator<T: SceneDecorator + 'static>(&self) -> Option<&T> {
        self.decorators.iter()
            .find_map(|d| d.as_any().downcast_ref::<T>())
    }
}
```

### 2.4 Handling Upstream Updates

```
┌─────────────────────────────────────────────────────────────┐
│              Upstream Update Workflow                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   1. Fetch upstream changes                                  │
│      git fetch upstream && git diff upstream/main           │
│                                                              │
│   2. Run transpiler on new Python code                       │
│      py2cura-rs transpile --input=cura/ --output=cura-rs/   │
│                                                              │
│   3. Generate diff report                                    │
│      py2cura-rs diff --old=v5.11 --new=v5.12                │
│                                                              │
│   4. Apply manual patches from patch queue                   │
│      py2cura-rs apply-patches --patches=patches/            │
│                                                              │
│   5. Run test suite                                          │
│      cargo test --workspace                                  │
│                                                              │
│   6. Manual review of flagged sections                       │
│      (complex transformations, new patterns)                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Patch Queue System**: Manual fixes are stored as patches that can be re-applied:

```
patches/
├── 0001-fix-container-stack-lifetime.patch
├── 0002-optimize-settings-lookup.patch
├── 0003-add-async-to-file-io.patch
└── manifest.toml  # Patch metadata and application order
```

---

## 3. Core Module Mapping

### 3.1 Module-by-Module Translation Plan

| Python Module | Rust Crate | LOC | Priority | Notes |
|---------------|------------|-----|----------|-------|
| `cura/Settings/` | `cura-settings` | ~3,000 | P0 | Foundation for everything |
| `cura/Scene/` | `cura-scene` | ~500 | P0 | Model representation |
| `cura/Machines/` | `cura-machines` | ~1,500 | P0 | Printer/material hierarchy |
| `plugins/CuraEngineBackend/` | `cura-engine-bridge` | ~5,000 | P0 | Core slicing logic |
| `plugins/3MFReader/` | `cura-formats` | ~2,000 | P1 | File I/O |
| `plugins/GCodeReader/` | `cura-formats` | ~1,000 | P1 | File I/O |
| `cura/PrinterOutput/` | `cura-devices` | ~1,500 | P2 | Device communication |
| `cura/Arranging/` | `cura-arrange` | ~1,000 | P2 | Part placement |
| `cura/API/` | `cura-api` | ~500 | P1 | Public API surface |
| `cura/Operations/` | `cura-ops` | ~500 | P3 | Undo/redo (UI-focused) |
| `cura/OAuth2/` | `cura-auth` | ~800 | P3 | Cloud auth |
| `cura/UltimakerCloud/` | `cura-cloud` | ~1,500 | P3 | Cloud integration |

### 3.2 Settings System Deep Dive

The settings system is the most complex and critical subsystem:

```
Python Structure:
├── CuraContainerStack.py      → ContainerStack struct
├── GlobalStack.py             → GlobalStack struct
├── ExtruderStack.py           → ExtruderStack struct
├── MachineManager.py          → MachineManager service
├── ExtruderManager.py         → ExtruderManager service
├── CuraContainerRegistry.py   → ContainerRegistry service
├── ContainerManager.py        → ContainerManager service
├── SettingInheritanceManager  → SettingResolver
└── IntentManager.py           → IntentManager service
```

**Rust Design**:

```rust
// cura-settings/src/lib.rs

/// The container stack hierarchy for settings resolution
#[derive(Debug)]
pub struct ContainerStack {
    definition: Arc<DefinitionContainer>,
    definition_changes: InstanceContainer,
    quality: InstanceContainer,
    quality_changes: InstanceContainer,
    material: InstanceContainer,
    variant: InstanceContainer,
    intent: InstanceContainer,
    user: InstanceContainer,
}

impl ContainerStack {
    /// Resolve a setting value through the stack hierarchy
    pub fn get_property<T: SettingValue>(&self, key: &str) -> Option<T> {
        // Walk stack from user -> definition
        self.user.get(key)
            .or_else(|| self.intent.get(key))
            .or_else(|| self.variant.get(key))
            .or_else(|| self.material.get(key))
            .or_else(|| self.quality_changes.get(key))
            .or_else(|| self.quality.get(key))
            .or_else(|| self.definition_changes.get(key))
            .or_else(|| self.definition.get_default(key))
    }
}

/// Setting definition from .def.json files
#[derive(Debug, Clone, Deserialize)]
pub struct SettingDefinition {
    pub key: String,
    pub label: String,
    pub description: String,
    #[serde(rename = "type")]
    pub value_type: SettingType,
    pub default_value: SettingValue,
    pub unit: Option<String>,
    pub minimum_value: Option<f64>,
    pub maximum_value: Option<f64>,
    pub settable_per_mesh: bool,
    pub settable_per_extruder: bool,
    pub enabled: Option<String>, // Formula
    pub value: Option<String>,   // Computed formula
}

#[derive(Debug, Clone)]
pub enum SettingType {
    Int,
    Float,
    Bool,
    String,
    Enum(Vec<String>),
    Category,
    Polygon,
    Polygons,
}
```

---

## 4. Rust Crate Dependencies

### 4.1 Core Dependencies Mapping

| Python Dependency | Purpose | Rust Equivalent | Notes |
|-------------------|---------|-----------------|-------|
| **uranium** | 3D engine, settings | Custom (`cura-*`) | Port core abstractions |
| **pysavitar** | 3MF support | `zip` + `quick-xml` | Implement 3MF spec |
| **pynest2d** | Part nesting | `geo` + custom | Port nesting algorithm |
| **trimesh** | Mesh operations | `parry3d`, `nalgebra` | Robust Rust alternatives |
| **shapely** | Geometry | `geo`, `geos` bindings | Mature ecosystem |
| **numpy** | Numerics | `ndarray`, `nalgebra` | Native Rust performance |
| **protobuf** | CuraEngine comm | `prost`, `tonic` | Modern protobuf stack |
| **PyQt6** | UI framework | N/A (API-first) | Replaced by Tauri |
| **requests** | HTTP client | `reqwest` | Async-first |
| **zeroconf** | mDNS discovery | `mdns-sd` | Pure Rust |
| **keyring** | Credential storage | `keyring` | Cross-platform |
| **pyserial** | Serial comm | `serialport` | Active maintenance |
| **pyyaml** | YAML parsing | `serde_yaml` | Serde ecosystem |
| **sentry-sdk** | Error tracking | N/A (skip) | Not porting |

### 4.2 Complete Cargo.toml Dependencies

```toml
# cura-core/Cargo.toml

[package]
name = "cura-core"
version = "0.1.0"
edition = "2021"

[dependencies]
# Async runtime
tokio = { version = "1", features = ["full"] }
async-trait = "0.1"

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"

# Geometry and math
nalgebra = "0.32"
parry3d = "0.13"
geo = "0.26"

# Mesh handling
tobj = "4"  # OBJ/STL loading
gltf = "1"  # GLTF support

# 3MF format
zip = "0.6"
quick-xml = "0.31"

# Protobuf (CuraEngine communication)
prost = "0.12"
tonic = "0.10"

# HTTP client
reqwest = { version = "0.11", features = ["json"] }

# mDNS discovery
mdns-sd = "0.10"

# Serial communication
serialport = "4"

# Credential storage
keyring = "2"

# Configuration
config = "0.13"
directories = "5"

# Error handling
thiserror = "1"
anyhow = "1"

# Logging
tracing = "0.1"
tracing-subscriber = "0.3"

# Concurrency
parking_lot = "0.12"
once_cell = "1"
crossbeam-channel = "0.5"

# UUID generation
uuid = { version = "1", features = ["v4"] }

[build-dependencies]
prost-build = "0.12"
```

---

## 5. API Design

### 5.1 API Architecture

The API layer enables headless operation and multiple frontends:

```
┌─────────────────────────────────────────────────────────────────┐
│                         cura-api crate                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                    Transport Layer                          ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐ ││
│  │  │   REST   │  │   gRPC   │  │WebSocket │  │    MCP     │ ││
│  │  │  (axum)  │  │ (tonic)  │  │  (tokio) │  │ (mcp-rs)   │ ││
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬──────┘ ││
│  │       └─────────────┼─────────────┼──────────────┘        ││
│  └─────────────────────┼─────────────┼───────────────────────┘│
│                        ▼             ▼                         │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                   Service Layer                             ││
│  │  ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐ ││
│  │  │ SlicingService│ │ SceneService │ │ MachineService     │ ││
│  │  ├──────────────┤ ├──────────────┤ ├────────────────────┤ ││
│  │  │ • slice()    │ │ • load_model │ │ • list_machines    │ ││
│  │  │ • cancel()   │ │ • transform  │ │ • get_machine      │ ││
│  │  │ • status()   │ │ • arrange    │ │ • set_active       │ ││
│  │  └──────────────┘ └──────────────┘ └────────────────────┘ ││
│  │  ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐ ││
│  │  │MaterialService│ │ProfileService│ │ ExportService      │ ││
│  │  ├──────────────┤ ├──────────────┤ ├────────────────────┤ ││
│  │  │ • list       │ │ • list       │ │ • to_gcode         │ ││
│  │  │ • get        │ │ • get        │ │ • to_3mf           │ ││
│  │  │ • set_active │ │ • apply      │ │ • to_ufp           │ ││
│  │  └──────────────┘ └──────────────┘ └────────────────────┘ ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                   Event System                              ││
│  │  • Server-Sent Events (SSE) for progress                   ││
│  │  • WebSocket for real-time updates                         ││
│  │  • Broadcast channels for internal pub/sub                 ││
│  └────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 REST API Specification

```yaml
openapi: 3.0.0
info:
  title: Cura Core API
  version: 1.0.0

paths:
  # Machine Management
  /machines:
    get:
      summary: List available machine definitions
      responses:
        200:
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/MachineDefinition'

  /machines/active:
    get:
      summary: Get active machine configuration
    put:
      summary: Set active machine
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                machine_id: { type: string }

  # Scene Management
  /scene:
    get:
      summary: Get current scene state
    delete:
      summary: Clear the scene

  /scene/models:
    post:
      summary: Add model to scene
      requestBody:
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                file: { type: string, format: binary }
                transform: { $ref: '#/components/schemas/Transform' }

  /scene/models/{id}:
    get:
      summary: Get model details
    patch:
      summary: Update model transform/settings
    delete:
      summary: Remove model from scene

  /scene/arrange:
    post:
      summary: Auto-arrange models on build plate

  # Slicing
  /slice:
    post:
      summary: Start slicing job
      responses:
        202:
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SliceJob'

  /slice/{job_id}:
    get:
      summary: Get slicing job status
    delete:
      summary: Cancel slicing job

  /slice/{job_id}/gcode:
    get:
      summary: Download generated G-code
      responses:
        200:
          content:
            application/octet-stream: {}

  # Settings
  /settings:
    get:
      summary: Get all resolved settings
    patch:
      summary: Update user settings

  /settings/{key}:
    get:
      summary: Get specific setting value and metadata

  # Materials
  /materials:
    get:
      summary: List available materials

  /materials/active:
    get:
      summary: Get active material per extruder
    put:
      summary: Set active material

  # Profiles
  /profiles/quality:
    get:
      summary: List quality profiles

  /profiles/quality/active:
    put:
      summary: Set active quality profile

  # Export
  /export/gcode:
    post:
      summary: Export current slice as G-code file

  /export/3mf:
    post:
      summary: Export scene as 3MF project

  # Events (SSE)
  /events:
    get:
      summary: Server-Sent Events stream
      responses:
        200:
          content:
            text/event-stream: {}

components:
  schemas:
    MachineDefinition:
      type: object
      properties:
        id: { type: string }
        name: { type: string }
        manufacturer: { type: string }
        build_volume: { $ref: '#/components/schemas/BuildVolume' }

    BuildVolume:
      type: object
      properties:
        width: { type: number }
        depth: { type: number }
        height: { type: number }

    Transform:
      type: object
      properties:
        position: { type: array, items: { type: number } }
        rotation: { type: array, items: { type: number } }
        scale: { type: array, items: { type: number } }

    SliceJob:
      type: object
      properties:
        id: { type: string }
        status: { type: string, enum: [pending, running, completed, failed, cancelled] }
        progress: { type: number }
        estimated_time: { type: number }
        material_usage: { type: object }
```

### 5.3 MCP (Model Context Protocol) Integration

For AI/LLM interaction, expose tools via MCP:

```rust
// cura-api/src/mcp.rs

use mcp_rs::{Tool, ToolResult};

pub struct CuraMcpServer {
    core: Arc<CuraCore>,
}

impl CuraMcpServer {
    pub fn tools(&self) -> Vec<Tool> {
        vec![
            Tool::new("load_model")
                .description("Load a 3D model file into the scene")
                .parameter("file_path", "Path to STL/3MF/OBJ file")
                .parameter("auto_arrange", "Whether to auto-arrange after loading"),

            Tool::new("slice")
                .description("Slice the current scene and generate G-code")
                .parameter("output_path", "Where to save the G-code"),

            Tool::new("set_machine")
                .description("Set the active 3D printer")
                .parameter("machine_id", "Machine definition ID"),

            Tool::new("set_quality")
                .description("Set print quality profile")
                .parameter("quality", "Quality level: draft, normal, fine"),

            Tool::new("set_material")
                .description("Set the active material")
                .parameter("material_id", "Material ID")
                .parameter("extruder", "Extruder number (0-indexed)"),

            Tool::new("adjust_setting")
                .description("Modify a specific print setting")
                .parameter("key", "Setting key (e.g., layer_height)")
                .parameter("value", "New value"),

            Tool::new("get_scene_info")
                .description("Get information about models in the scene"),

            Tool::new("get_slice_preview")
                .description("Get print time and material estimates")
                .parameter("layer", "Optional specific layer to preview"),

            Tool::new("export_project")
                .description("Export scene as 3MF project file")
                .parameter("output_path", "Where to save the project"),
        ]
    }
}
```

---

## 6. Frontend Architecture

### 6.1 Tauri + Bun Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    cura-ui (Tauri + Bun Project)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │                      Web Layer (Bun + React)                    ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐   ││
│  │  │   3D Viewer  │  │   Settings   │  │    Print Preview   │   ││
│  │  │  (Three.js)  │  │    Panel     │  │   (Layer Viz)      │   ││
│  │  └──────────────┘  └──────────────┘  └────────────────────┘   ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐   ││
│  │  │   Machine    │  │   Material   │  │    Print Queue     │   ││
│  │  │   Selector   │  │   Browser    │  │    & Monitor       │   ││
│  │  └──────────────┘  └──────────────┘  └────────────────────┘   ││
│  └────────────────────────────────────────────────────────────────┘│
│                               │                                      │
│                               │ Tauri IPC / HTTP                     │
│                               ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │                    Tauri Rust Backend                           ││
│  │  ┌──────────────────────────────────────────────────────────┐ ││
│  │  │  Option A: Embedded cura-core (single binary)            │ ││
│  │  │  Option B: HTTP client to standalone cura-api server     │ ││
│  │  └──────────────────────────────────────────────────────────┘ ││
│  │                                                                ││
│  │  • Native file dialogs                                        ││
│  │  • System tray integration                                    ││
│  │  • Deep OS integration                                        ││
│  │  • Auto-update mechanism                                      ││
│  └────────────────────────────────────────────────────────────────┘│
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 Project Structure

```
cura-ui/
├── src-tauri/              # Tauri Rust backend
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs
│   │   ├── commands.rs     # Tauri IPC commands
│   │   └── state.rs        # Application state
│   └── tauri.conf.json
│
├── src/                    # Bun/React frontend
│   ├── index.html
│   ├── main.tsx
│   ├── App.tsx
│   ├── components/
│   │   ├── viewer/         # 3D viewport (Three.js)
│   │   │   ├── SceneViewer.tsx
│   │   │   ├── BuildPlate.tsx
│   │   │   └── ModelRenderer.tsx
│   │   ├── settings/       # Settings panel
│   │   │   ├── SettingsPanel.tsx
│   │   │   ├── SettingInput.tsx
│   │   │   └── CategoryAccordion.tsx
│   │   ├── machines/       # Machine selection
│   │   ├── materials/      # Material browser
│   │   ├── preview/        # Slice preview
│   │   └── common/         # Shared components
│   ├── hooks/
│   │   ├── useCuraApi.ts   # API client hook
│   │   ├── useScene.ts
│   │   └── useSlicing.ts
│   ├── stores/             # State management (Zustand)
│   │   ├── sceneStore.ts
│   │   ├── machineStore.ts
│   │   └── settingsStore.ts
│   └── api/
│       └── client.ts       # Generated API client
│
├── package.json
├── bun.lockb
├── tsconfig.json
└── vite.config.ts
```

### 6.3 Multi-Platform Build

```bash
# Desktop (Tauri)
bun run tauri build          # Builds for current platform
bun run tauri build --target universal-apple-darwin  # macOS universal
bun run tauri build --target x86_64-pc-windows-msvc  # Windows

# Web (standalone)
bun run build:web            # Static site connecting to API server

# Development
bun run tauri dev            # Desktop with hot reload
bun run dev:web              # Web-only development
```

### 6.4 API Client Generation

Use OpenAPI to generate TypeScript client:

```typescript
// Generated from OpenAPI spec
import { Configuration, DefaultApi } from './generated';

const config = new Configuration({
  basePath: window.__TAURI__
    ? 'tauri://localhost'  // IPC in desktop
    : 'http://localhost:8080'  // HTTP in web
});

export const curaApi = new DefaultApi(config);

// Usage
const machines = await curaApi.getMachines();
const sliceJob = await curaApi.startSlice();
```

---

## 7. Implementation Phases

### Phase 0: Tooling Foundation (4-6 weeks)

**Goal**: Build the programmatic porting infrastructure

```
Week 1-2: Parser Setup
├── Set up py2cura-rs project structure
├── Integrate RustPython parser for AST
├── Define intermediate representation (IR)
└── Basic Python → IR transformation

Week 3-4: Core Transforms
├── Type inference engine
├── Python → Rust type mapping
├── Signal → channel transformation
├── Decorator → trait transformation

Week 5-6: Code Generation
├── IR → Rust code generator (syn/quote)
├── Post-processing (cargo fmt, clippy fixes)
├── Patch system for manual overrides
└── Diff tracking for upstream changes
```

**Deliverables**:
- `py2cura-rs` CLI tool
- Initial transformation rules
- Patch management system

### Phase 1: Core Library (8-10 weeks)

**Goal**: Port essential modules to functional Rust

```
Week 1-3: cura-settings
├── ContainerStack implementation
├── DefinitionContainer parser (.def.json)
├── InstanceContainer (.inst.cfg)
├── Setting resolution algorithm
└── Setting formula evaluation

Week 4-5: cura-scene
├── SceneNode and decorators
├── Mesh representation
├── Transform system
├── Build plate management

Week 6-7: cura-machines
├── Machine definition loading
├── ContainerTree hierarchy
├── Material/Quality resolution
├── Variant handling

Week 8-10: cura-engine-bridge
├── Protobuf message generation
├── StartSliceJob logic port
├── Engine process management
├── G-code post-processing
```

**Deliverables**:
- `cura-settings`, `cura-scene`, `cura-machines`, `cura-engine-bridge` crates
- Integration tests with CuraEngine
- Basic CLI for validation

### Phase 2: API Layer (4-6 weeks)

**Goal**: Expose core functionality via API

```
Week 1-2: REST API
├── axum server setup
├── Machine/Material/Profile endpoints
├── Scene management endpoints
├── Settings endpoints

Week 3-4: Slicing API
├── Async slicing jobs
├── Progress streaming (SSE)
├── G-code download
├── Job cancellation

Week 5-6: MCP Integration
├── MCP server implementation
├── Tool definitions
├── Resource exposures
└── Prompts for common workflows
```

**Deliverables**:
- Running API server
- OpenAPI specification
- MCP server with slicing tools

### Phase 3: File I/O (3-4 weeks)

**Goal**: Support all major file formats

```
Week 1-2: 3MF Support
├── 3MF reader (zip + XML)
├── 3MF writer
├── Workspace format (full project)

Week 3-4: Other Formats
├── G-code reader
├── STL/OBJ import (via tobj/parry3d)
├── Profile import/export
```

**Deliverables**:
- `cura-formats` crate
- Round-trip 3MF support
- Profile migration tools

### Phase 4: UI Development (6-8 weeks)

**Goal**: Functional Tauri desktop application

```
Week 1-2: Project Setup
├── Tauri + Bun + React scaffolding
├── Build system configuration
├── API client generation

Week 3-4: 3D Viewer
├── Three.js scene setup
├── Model loading and rendering
├── Build plate visualization
├── Interaction (rotate, pan, zoom)

Week 5-6: Settings & Machine UI
├── Settings panel with categories
├── Machine selector
├── Material browser
├── Quality selector

Week 7-8: Slicing & Preview
├── Slice button and progress
├── Layer visualization
├── Print time/material display
├── G-code export
```

**Deliverables**:
- Functional desktop application
- Web deployment option
- Feature parity with core Cura workflows

### Phase 5: Device Integration (4-6 weeks)

**Goal**: Printer connectivity

```
Week 1-2: Discovery
├── mDNS printer discovery
├── USB detection
├── Device registry

Week 3-4: Communication
├── Network printer protocol
├── USB serial communication
├── Print job management

Week 5-6: Monitoring
├── Print progress tracking
├── Printer status updates
├── Camera feeds (if applicable)
```

**Deliverables**:
- `cura-devices` crate
- Network and USB printing
- Print monitoring

### Phase 6: Polish & Release (4-6 weeks)

**Goal**: Production-ready release

```
├── Comprehensive testing
├── Performance optimization
├── Documentation
├── Packaging and distribution
├── CI/CD pipeline
└── Community feedback incorporation
```

---

## 8. Tooling Specification

### 8.1 py2cura-rs Transpiler

```
py2cura-rs/
├── Cargo.toml
├── src/
│   ├── main.rs              # CLI entry point
│   ├── lib.rs
│   ├── parser/
│   │   ├── mod.rs
│   │   └── python_ast.rs    # RustPython AST integration
│   ├── ir/
│   │   ├── mod.rs
│   │   ├── types.rs         # IR type system
│   │   ├── expressions.rs
│   │   └── statements.rs
│   ├── transform/
│   │   ├── mod.rs
│   │   ├── type_inference.rs
│   │   ├── signal_to_channel.rs
│   │   ├── decorator_to_trait.rs
│   │   ├── singleton.rs
│   │   └── cura_specific.rs # Domain-specific transforms
│   ├── codegen/
│   │   ├── mod.rs
│   │   └── rust_emitter.rs  # syn/quote code generation
│   └── patches/
│       ├── mod.rs
│       └── apply.rs         # Patch application
├── patches/                  # Manual fix patches
│   └── manifest.toml
└── tests/
    └── integration/
```

### 8.2 CLI Interface

```bash
# Full transpilation
py2cura-rs transpile \
  --input ./cura \
  --output ./cura-rs \
  --config ./transpile.toml

# Differential update (upstream changes)
py2cura-rs update \
  --upstream-ref v5.12.0 \
  --output ./cura-rs

# Apply patches
py2cura-rs patch \
  --patches ./patches \
  --target ./cura-rs

# Generate diff report
py2cura-rs diff \
  --old v5.11.0 \
  --new v5.12.0 \
  --output diff-report.md
```

### 8.3 Configuration

```toml
# transpile.toml

[general]
source_version = "5.11.0"
target_edition = "2021"

[type_mappings]
"PyQt6.QtCore.pyqtSignal" = "tokio::sync::broadcast::Sender"
"UM.Signal.Signal" = "tokio::sync::broadcast::Sender"
"UM.Scene.SceneNode" = "cura_scene::SceneNode"

[module_mappings]
"cura.Settings" = "cura_settings"
"cura.Scene" = "cura_scene"
"cura.Machines" = "cura_machines"

[skip_modules]
# UI-only modules, don't transpile
patterns = [
  "cura/UI/*",
  "resources/qml/*",
  "plugins/*/qml/*"
]

[manual_overrides]
# Files with custom implementations
"cura/Settings/MachineManager.py" = "manual/machine_manager.rs"

[patches]
directory = "./patches"
auto_apply = true
```

---

## Appendix A: Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| CuraEngine protocol changes | High | Version lock, maintain protocol adapters |
| Complex Python dynamics | Medium | Comprehensive test suite, manual review |
| Settings formula evaluation | High | Port formula parser, extensive testing |
| Plugin ecosystem | Medium | Define stable plugin API, gradual migration |
| Performance regression | Medium | Benchmark suite, profile-guided optimization |
| Upstream divergence | Medium | Regular sync cadence, automated diff reports |

## Appendix B: Success Metrics

- [ ] 100% settings resolution parity with Python Cura
- [ ] G-code output byte-identical for same inputs
- [ ] API response time < 100ms for common operations
- [ ] Slice times within 10% of native CuraEngine CLI
- [ ] Memory usage < Python version
- [ ] Desktop app startup < 2 seconds
- [ ] Successful upstream sync on each minor version

## Appendix C: Resource Estimates

| Phase | Duration | Engineers |
|-------|----------|-----------|
| Phase 0: Tooling | 4-6 weeks | 1-2 |
| Phase 1: Core | 8-10 weeks | 2-3 |
| Phase 2: API | 4-6 weeks | 1-2 |
| Phase 3: File I/O | 3-4 weeks | 1 |
| Phase 4: UI | 6-8 weeks | 2-3 |
| Phase 5: Devices | 4-6 weeks | 1-2 |
| Phase 6: Polish | 4-6 weeks | 2-3 |
| **Total** | **33-46 weeks** | **2-3 avg** |

---

*Document Version: 1.0*
*Last Updated: 2026-02-05*
