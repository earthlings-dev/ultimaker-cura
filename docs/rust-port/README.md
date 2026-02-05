# Cura Rust Port Planning

This directory contains comprehensive planning documents for porting Ultimaker Cura from Python/PyQt to Rust with an API-first architecture.

## Documents

| Document | Description |
|----------|-------------|
| [PORTING_PLAN.md](./PORTING_PLAN.md) | Master plan covering architecture, phases, and timeline |
| [TRANSPILER_SPEC.md](./TRANSPILER_SPEC.md) | Specification for the `py2cura-rs` programmatic transpiler |
| [DEPENDENCY_MAPPING.md](./DEPENDENCY_MAPPING.md) | Python → Rust crate equivalents for all dependencies |
| [API_ARCHITECTURE.md](./API_ARCHITECTURE.md) | REST/WebSocket/MCP API design for headless operation |
| [UI_ARCHITECTURE.md](./UI_ARCHITECTURE.md) | Tauri + Bun frontend architecture |

## Quick Overview

### Goals

1. **API-First**: Core slicer logic exposed via REST/WebSocket/MCP APIs
2. **Headless Operation**: Run without a GUI for automation and AI integration
3. **Programmatic Porting**: Automated transpilation from Python to Rust
4. **Upstream Tracking**: Re-runnable tooling when new Cura versions release
5. **Modern UI**: Separate Tauri + Bun project for cross-platform desktop/web

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    FRONTENDS                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐│
│  │  Tauri   │  │   Web    │  │   MCP    │  │   CLI   ││
│  │ Desktop  │  │ Browser  │  │ (Claude) │  │ Scripts ││
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬────┘│
└───────┼─────────────┼─────────────┼─────────────┼──────┘
        └─────────────┴─────────────┴─────────────┘
                          │
                          ▼
         ┌────────────────────────────────┐
         │     cura-api (REST/WS/MCP)     │
         └────────────────┬───────────────┘
                          │
         ┌────────────────▼───────────────┐
         │         cura-core (Rust)        │
         │  ┌─────────┐ ┌─────────┐       │
         │  │settings │ │ scene   │       │
         │  ├─────────┤ ├─────────┤       │
         │  │machines │ │ formats │       │
         │  ├─────────┤ ├─────────┤       │
         │  │ engine  │ │ devices │       │
         │  └─────────┘ └─────────┘       │
         └────────────────┬───────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  CuraEngine (C++)     │
              │  (Unchanged)          │
              └───────────────────────┘
```

### Key Decisions

1. **Keep CuraEngine**: The C++ slicing backend remains unchanged; only the Python orchestration layer is ported
2. **Programmatic Transpilation**: Build `py2cura-rs` tool for automated conversion with manual patch support
3. **API-First Design**: All functionality accessible via API before any UI is built
4. **Separate UI Project**: UI in its own repository using Tauri + Bun for desktop/web
5. **MCP Integration**: First-class support for AI assistant interaction

### Implementation Phases

| Phase | Focus | Duration |
|-------|-------|----------|
| 0 | Transpiler tooling (`py2cura-rs`) | 4-6 weeks |
| 1 | Core library (settings, scene, machines, engine bridge) | 8-10 weeks |
| 2 | API layer (REST, WebSocket, MCP) | 4-6 weeks |
| 3 | File I/O (3MF, G-code, profiles) | 3-4 weeks |
| 4 | UI development (Tauri + Bun) | 6-8 weeks |
| 5 | Device integration (network, USB) | 4-6 weeks |
| 6 | Polish and release | 4-6 weeks |

**Total estimated duration: 33-46 weeks**

### Core Rust Crates

| Crate | Purpose |
|-------|---------|
| `cura-settings` | Container stack, setting resolution |
| `cura-scene` | Scene graph, model management |
| `cura-machines` | Machine/material/quality hierarchy |
| `cura-engine-bridge` | CuraEngine protobuf communication |
| `cura-formats` | 3MF, G-code, profile parsing |
| `cura-devices` | Printer discovery and communication |
| `cura-api` | REST/WebSocket/MCP server |
| `cura-core` | Workspace crate tying everything together |

### Getting Started

These documents are the planning phase. Implementation would begin with:

1. **Set up Rust workspace** with the crate structure above
2. **Build `py2cura-rs`** transpiler MVP targeting `cura/Settings/`
3. **Port settings system** manually with transpiler assistance
4. **Add CuraEngine bridge** for basic slicing
5. **Build API server** exposing core functionality
6. **Develop UI** in parallel once API is stable

## Contributing

This is a planning document. To contribute:

1. Review the documents and identify gaps
2. Open issues for discussion
3. Submit PRs to improve the plans

## License

Same as Cura: LGPL-3.0-or-later
