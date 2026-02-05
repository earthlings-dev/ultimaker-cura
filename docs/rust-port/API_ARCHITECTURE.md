# Cura Core API Architecture

## Overview

The API-first architecture enables Cura to run headlessly and be accessed from multiple clients: desktop (Tauri), web browsers, CLI tools, and AI assistants via MCP (Model Context Protocol).

## Design Principles

1. **Stateful Service**: The API maintains scene state, settings, and machine configuration
2. **Async-First**: All potentially long operations return job handles
3. **Event-Driven**: Real-time updates via WebSocket/SSE
4. **Resource-Oriented**: RESTful design for CRUD operations
5. **MCP-Compatible**: Exposed as tools for AI assistants

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Clients                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐│
│  │  Desktop    │  │   Web UI    │  │    CLI      │  │   MCP Client        ││
│  │  (Tauri)    │  │  (Browser)  │  │  (cura-cli) │  │ (Claude, etc.)      ││
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘│
│         │                │                │                     │           │
│         │ Tauri IPC      │ HTTP/WS        │ HTTP               │ MCP       │
│         │                │                │                     │           │
└─────────┼────────────────┼────────────────┼─────────────────────┼───────────┘
          │                │                │                     │
          ▼                ▼                ▼                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           cura-api Server                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │                        Transport Layer                                  ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ ││
│  │  │    REST      │  │  WebSocket   │  │     SSE      │  │    MCP     │ ││
│  │  │   (axum)     │  │   (tokio)    │  │   (axum)     │  │  (mcp-rs)  │ ││
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ ││
│  │         │                 │                 │                │        ││
│  │         └─────────────────┼─────────────────┼────────────────┘        ││
│  │                           │                 │                          ││
│  │                           ▼                 ▼                          ││
│  └───────────────────────────┬─────────────────┬──────────────────────────┘│
│                              │                 │                            │
│  ┌───────────────────────────▼─────────────────▼──────────────────────────┐│
│  │                       Service Layer                                     ││
│  │                                                                         ││
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐   ││
│  │  │ SceneService   │  │ SlicingService │  │ MachineService         │   ││
│  │  ├────────────────┤  ├────────────────┤  ├────────────────────────┤   ││
│  │  │ • load_model   │  │ • start_slice  │  │ • list_definitions     │   ││
│  │  │ • transform    │  │ • cancel       │  │ • activate_machine     │   ││
│  │  │ • remove       │  │ • get_status   │  │ • get_active           │   ││
│  │  │ • arrange      │  │ • get_gcode    │  │ • get_settings         │   ││
│  │  │ • clear        │  │               │  │                         │   ││
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘   ││
│  │                                                                         ││
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐   ││
│  │  │ MaterialService│  │ ProfileService │  │ DeviceService          │   ││
│  │  ├────────────────┤  ├────────────────┤  ├────────────────────────┤   ││
│  │  │ • list         │  │ • list_quality │  │ • discover             │   ││
│  │  │ • activate     │  │ • apply_quality│  │ • connect              │   ││
│  │  │ • get_active   │  │ • save_profile │  │ • print                │   ││
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘   ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                      │                                      │
│  ┌───────────────────────────────────▼─────────────────────────────────────┐│
│  │                          Core Layer                                      ││
│  │                                                                          ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ ││
│  │  │ cura-scene   │  │cura-settings │  │cura-machines │  │cura-formats │ ││
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └─────────────┘ ││
│  │                                                                          ││
│  │  ┌───────────────────────────────┐  ┌──────────────────────────────────┐││
│  │  │     cura-engine-bridge        │  │         cura-devices             │││
│  │  └───────────────────────────────┘  └──────────────────────────────────┘││
│  └──────────────────────────────────────────────────────────────────────────┘│
│                                      │                                       │
└──────────────────────────────────────┼───────────────────────────────────────┘
                                       │
                                       ▼
                        ┌──────────────────────────────┐
                        │      CuraEngine (C++)        │
                        │  (Unchanged - Protobuf IPC)  │
                        └──────────────────────────────┘
```

---

## REST API Specification

### Base URL

```
http://localhost:8080/api/v1
```

### Authentication

For local use, no authentication required. For remote access:

```http
Authorization: Bearer <token>
```

### Common Response Format

```json
{
  "data": { ... },      // Response payload
  "error": null,        // Error object if failed
  "meta": {
    "request_id": "uuid",
    "timestamp": "ISO8601"
  }
}
```

### Error Response

```json
{
  "data": null,
  "error": {
    "code": "INVALID_MODEL",
    "message": "Failed to parse STL file",
    "details": { ... }
  },
  "meta": { ... }
}
```

---

## Endpoints

### Scene Management

#### GET /scene
Get current scene state.

**Response:**
```json
{
  "data": {
    "models": [
      {
        "id": "uuid",
        "name": "model.stl",
        "path": "/path/to/model.stl",
        "transform": {
          "position": [0, 0, 0],
          "rotation": [0, 0, 0, 1],
          "scale": [1, 1, 1]
        },
        "bounding_box": {
          "min": [-10, -10, 0],
          "max": [10, 10, 20]
        },
        "settings_override": {
          "infill_sparse_density": 30
        }
      }
    ],
    "build_plate": {
      "width": 235,
      "depth": 235,
      "shape": "rectangular"
    }
  }
}
```

#### POST /scene/models
Add a model to the scene.

**Request (multipart/form-data):**
```
file: <binary>
auto_arrange: true
```

**Response:**
```json
{
  "data": {
    "id": "uuid",
    "name": "model.stl",
    "bounding_box": { ... }
  }
}
```

#### PATCH /scene/models/{id}
Update model transform or settings.

**Request:**
```json
{
  "transform": {
    "position": [10, 20, 0],
    "rotation": [0, 0, 0.707, 0.707]
  },
  "settings_override": {
    "support_enable": true
  }
}
```

#### DELETE /scene/models/{id}
Remove model from scene.

#### POST /scene/arrange
Auto-arrange all models.

**Request:**
```json
{
  "algorithm": "nest2d",
  "spacing": 5.0
}
```

#### DELETE /scene
Clear all models from scene.

---

### Machine Management

#### GET /machines
List available machine definitions.

**Query Parameters:**
- `manufacturer`: Filter by manufacturer
- `visible`: Include only visible machines (default: true)

**Response:**
```json
{
  "data": [
    {
      "id": "ultimaker_s5",
      "name": "Ultimaker S5",
      "manufacturer": "Ultimaker",
      "build_volume": {
        "width": 330,
        "depth": 240,
        "height": 300
      },
      "extruder_count": 2
    }
  ]
}
```

#### GET /machines/active
Get currently active machine configuration.

**Response:**
```json
{
  "data": {
    "definition": {
      "id": "ultimaker_s5",
      "name": "Ultimaker S5"
    },
    "extruders": [
      {
        "index": 0,
        "material": {
          "id": "generic_pla",
          "name": "Generic PLA",
          "color": "#FFFFFF"
        },
        "variant": {
          "id": "AA 0.4",
          "nozzle_size": 0.4
        }
      }
    ],
    "quality_profile": {
      "id": "normal",
      "name": "Normal",
      "layer_height": 0.15
    }
  }
}
```

#### PUT /machines/active
Set the active machine.

**Request:**
```json
{
  "machine_id": "ultimaker_s5"
}
```

---

### Materials

#### GET /materials
List available materials for current machine.

**Query Parameters:**
- `extruder`: Filter by compatible extruder
- `type`: Filter by material type (pla, abs, petg, etc.)

**Response:**
```json
{
  "data": [
    {
      "id": "generic_pla",
      "name": "Generic PLA",
      "brand": "Generic",
      "type": "PLA",
      "color": "#FFFFFF",
      "compatible_variants": ["AA 0.4", "AA 0.8"]
    }
  ]
}
```

#### PUT /materials/active
Set active material for extruder.

**Request:**
```json
{
  "extruder": 0,
  "material_id": "generic_pla"
}
```

---

### Quality Profiles

#### GET /profiles/quality
List quality profiles for current machine/material combination.

**Response:**
```json
{
  "data": [
    {
      "id": "draft",
      "name": "Draft",
      "layer_height": 0.3,
      "print_speed": 70
    },
    {
      "id": "normal",
      "name": "Normal",
      "layer_height": 0.15,
      "print_speed": 50
    },
    {
      "id": "fine",
      "name": "Fine",
      "layer_height": 0.1,
      "print_speed": 40
    }
  ]
}
```

#### PUT /profiles/quality/active
Set active quality profile.

**Request:**
```json
{
  "profile_id": "normal"
}
```

---

### Settings

#### GET /settings
Get all resolved settings values.

**Query Parameters:**
- `category`: Filter by category (machine, quality, material, etc.)
- `keys`: Comma-separated list of specific keys

**Response:**
```json
{
  "data": {
    "layer_height": {
      "value": 0.15,
      "source": "quality",
      "label": "Layer Height",
      "unit": "mm",
      "type": "float"
    },
    "infill_sparse_density": {
      "value": 20,
      "source": "user",
      "label": "Infill Density",
      "unit": "%",
      "type": "int"
    }
  }
}
```

#### PATCH /settings
Update user settings.

**Request:**
```json
{
  "layer_height": 0.2,
  "infill_sparse_density": 30,
  "support_enable": true
}
```

#### GET /settings/{key}
Get single setting with full metadata.

**Response:**
```json
{
  "data": {
    "key": "layer_height",
    "value": 0.15,
    "default_value": 0.2,
    "source": "quality",
    "label": "Layer Height",
    "description": "The height of each printed layer",
    "type": "float",
    "unit": "mm",
    "minimum_value": 0.06,
    "maximum_value": 0.6,
    "settable_per_mesh": true,
    "settable_per_extruder": false
  }
}
```

---

### Slicing

#### POST /slice
Start a slicing job.

**Request:**
```json
{
  "models": ["uuid1", "uuid2"],  // Optional: specific models, default all
  "output_format": "gcode"       // gcode, ufp
}
```

**Response:**
```json
{
  "data": {
    "job_id": "uuid",
    "status": "pending",
    "created_at": "2026-02-05T10:30:00Z"
  }
}
```

#### GET /slice/{job_id}
Get slicing job status.

**Response (in progress):**
```json
{
  "data": {
    "job_id": "uuid",
    "status": "running",
    "progress": 0.45,
    "current_layer": 150,
    "total_layers": 330,
    "started_at": "2026-02-05T10:30:05Z"
  }
}
```

**Response (completed):**
```json
{
  "data": {
    "job_id": "uuid",
    "status": "completed",
    "progress": 1.0,
    "result": {
      "print_time": {
        "total_seconds": 7200,
        "formatted": "2h 0m"
      },
      "material_usage": [
        {
          "extruder": 0,
          "material": "Generic PLA",
          "length_mm": 15234.5,
          "weight_g": 45.7,
          "cost": 1.25
        }
      ],
      "layer_count": 330
    },
    "completed_at": "2026-02-05T10:35:00Z"
  }
}
```

#### DELETE /slice/{job_id}
Cancel a slicing job.

#### GET /slice/{job_id}/gcode
Download generated G-code.

**Response:** `application/octet-stream`

#### GET /slice/{job_id}/preview
Get layer preview data.

**Query Parameters:**
- `layer`: Specific layer number
- `type`: Preview type (paths, thumbnail)

---

### Export

#### POST /export/3mf
Export scene as 3MF file.

**Request:**
```json
{
  "include_settings": true,
  "include_thumbnails": true
}
```

**Response:** `application/octet-stream` (3MF file)

#### POST /export/gcode
Export last slice result as G-code file.

**Response:** `application/octet-stream`

---

### Devices

#### GET /devices
List discovered printers.

**Response:**
```json
{
  "data": [
    {
      "id": "192.168.1.100",
      "name": "Ultimaker S5 - Office",
      "type": "network",
      "status": "idle",
      "model": "Ultimaker S5",
      "firmware_version": "7.0.3"
    },
    {
      "id": "/dev/ttyUSB0",
      "name": "USB Printer",
      "type": "usb",
      "status": "connected"
    }
  ]
}
```

#### POST /devices/discover
Trigger printer discovery.

#### POST /devices/{id}/print
Send G-code to printer.

**Request:**
```json
{
  "job_id": "slice_job_uuid",
  "job_name": "My Print"
}
```

---

### Events (SSE)

#### GET /events
Server-Sent Events stream for real-time updates.

**Event Types:**

```
event: slice_progress
data: {"job_id": "uuid", "progress": 0.45, "layer": 150}

event: slice_completed
data: {"job_id": "uuid", "result": {...}}

event: scene_changed
data: {"action": "model_added", "model_id": "uuid"}

event: device_discovered
data: {"device": {...}}

event: settings_changed
data: {"keys": ["layer_height", "infill_sparse_density"]}
```

---

## WebSocket API

For bidirectional real-time communication:

```
ws://localhost:8080/ws
```

### Message Format

```json
{
  "type": "request|response|event",
  "id": "request_id",           // For request/response correlation
  "action": "slice.start",       // For requests
  "payload": { ... }
}
```

### Example Session

```json
// Client → Server: Start slice
{
  "type": "request",
  "id": "req_1",
  "action": "slice.start",
  "payload": {}
}

// Server → Client: Acknowledgment
{
  "type": "response",
  "id": "req_1",
  "payload": {"job_id": "uuid"}
}

// Server → Client: Progress events
{
  "type": "event",
  "action": "slice.progress",
  "payload": {"job_id": "uuid", "progress": 0.25}
}

// Server → Client: Completion
{
  "type": "event",
  "action": "slice.completed",
  "payload": {"job_id": "uuid", "result": {...}}
}
```

---

## MCP Integration

### Tool Definitions

```json
{
  "name": "cura",
  "version": "1.0.0",
  "description": "3D printing slicer for generating G-code",
  "tools": [
    {
      "name": "load_model",
      "description": "Load a 3D model file (STL, OBJ, 3MF) into the scene for slicing",
      "inputSchema": {
        "type": "object",
        "properties": {
          "file_path": {
            "type": "string",
            "description": "Absolute path to the 3D model file"
          },
          "auto_arrange": {
            "type": "boolean",
            "description": "Whether to auto-arrange models after loading",
            "default": true
          }
        },
        "required": ["file_path"]
      }
    },
    {
      "name": "slice",
      "description": "Slice the current scene to generate G-code for 3D printing",
      "inputSchema": {
        "type": "object",
        "properties": {
          "output_path": {
            "type": "string",
            "description": "Path where to save the generated G-code"
          }
        },
        "required": ["output_path"]
      }
    },
    {
      "name": "set_machine",
      "description": "Set the active 3D printer model for slicing",
      "inputSchema": {
        "type": "object",
        "properties": {
          "machine_id": {
            "type": "string",
            "description": "Machine definition ID (e.g., 'ultimaker_s5', 'creality_ender3')"
          }
        },
        "required": ["machine_id"]
      }
    },
    {
      "name": "set_quality",
      "description": "Set the print quality profile",
      "inputSchema": {
        "type": "object",
        "properties": {
          "quality": {
            "type": "string",
            "enum": ["draft", "normal", "fine"],
            "description": "Quality level for printing"
          }
        },
        "required": ["quality"]
      }
    },
    {
      "name": "set_material",
      "description": "Set the active material for printing",
      "inputSchema": {
        "type": "object",
        "properties": {
          "material_id": {
            "type": "string",
            "description": "Material ID (e.g., 'generic_pla', 'generic_petg')"
          },
          "extruder": {
            "type": "integer",
            "description": "Extruder index (0-based)",
            "default": 0
          }
        },
        "required": ["material_id"]
      }
    },
    {
      "name": "adjust_setting",
      "description": "Modify a specific print setting",
      "inputSchema": {
        "type": "object",
        "properties": {
          "key": {
            "type": "string",
            "description": "Setting key (e.g., 'layer_height', 'infill_sparse_density', 'support_enable')"
          },
          "value": {
            "type": ["number", "boolean", "string"],
            "description": "New value for the setting"
          }
        },
        "required": ["key", "value"]
      }
    },
    {
      "name": "get_scene_info",
      "description": "Get information about models currently in the scene"
    },
    {
      "name": "get_print_estimate",
      "description": "Get estimated print time and material usage for current scene"
    },
    {
      "name": "list_machines",
      "description": "List all available printer definitions"
    },
    {
      "name": "list_materials",
      "description": "List all available materials for the current machine"
    },
    {
      "name": "clear_scene",
      "description": "Remove all models from the scene"
    }
  ],
  "resources": [
    {
      "name": "scene",
      "description": "Current scene state with all models",
      "uri": "cura://scene"
    },
    {
      "name": "settings",
      "description": "Current print settings",
      "uri": "cura://settings"
    },
    {
      "name": "machine",
      "description": "Active machine configuration",
      "uri": "cura://machine"
    }
  ]
}
```

### Example MCP Session

```
User: "I have a 3D model at /home/user/bracket.stl. Can you slice it for my Ender 3 using PLA with 20% infill?"
AI Assistant: I'll help you slice that model. Let me:
1. Set up your Ender 3 printer
2. Configure PLA material
3. Adjust the infill to 20%
4. Load and slice your model

[Calls set_machine with machine_id="creality_ender3"]
[Calls set_material with material_id="generic_pla"]  
[Calls adjust_setting with key="infill_sparse_density", value=20]
[Calls load_model with file_path="/home/user/bracket.stl"]
[Calls slice with output_path="/home/user/bracket.gcode"]

Done\! Your G-code has been generated at /home/user/bracket.gcode.
Print time: 2h 15m
Material: 45g of PLA
```

---

## Rust Implementation

### Server Setup (axum)

```rust
use axum::{
    routing::{get, post, put, patch, delete},
    Router,
    extract::State,
};
use std::sync::Arc;
use tokio::sync::RwLock;

pub struct AppState {
    pub scene: RwLock<Scene>,
    pub machine_manager: RwLock<MachineManager>,
    pub slicing_service: SlicingService,
    pub event_broadcaster: broadcast::Sender<ServerEvent>,
}

pub fn create_router(state: Arc<AppState>) -> Router {
    Router::new()
        // Scene
        .route("/api/v1/scene", get(handlers::get_scene))
        .route("/api/v1/scene", delete(handlers::clear_scene))
        .route("/api/v1/scene/models", post(handlers::add_model))
        .route("/api/v1/scene/models/:id", get(handlers::get_model))
        .route("/api/v1/scene/models/:id", patch(handlers::update_model))
        .route("/api/v1/scene/models/:id", delete(handlers::remove_model))
        .route("/api/v1/scene/arrange", post(handlers::arrange_models))
        
        // Machines
        .route("/api/v1/machines", get(handlers::list_machines))
        .route("/api/v1/machines/active", get(handlers::get_active_machine))
        .route("/api/v1/machines/active", put(handlers::set_active_machine))
        
        // Materials
        .route("/api/v1/materials", get(handlers::list_materials))
        .route("/api/v1/materials/active", put(handlers::set_active_material))
        
        // Profiles
        .route("/api/v1/profiles/quality", get(handlers::list_quality_profiles))
        .route("/api/v1/profiles/quality/active", put(handlers::set_quality_profile))
        
        // Settings
        .route("/api/v1/settings", get(handlers::get_settings))
        .route("/api/v1/settings", patch(handlers::update_settings))
        .route("/api/v1/settings/:key", get(handlers::get_setting))
        
        // Slicing
        .route("/api/v1/slice", post(handlers::start_slice))
        .route("/api/v1/slice/:id", get(handlers::get_slice_status))
        .route("/api/v1/slice/:id", delete(handlers::cancel_slice))
        .route("/api/v1/slice/:id/gcode", get(handlers::download_gcode))
        
        // Events
        .route("/api/v1/events", get(handlers::events_sse))
        
        // WebSocket
        .route("/ws", get(handlers::websocket_handler))
        
        .with_state(state)
}

#[tokio::main]
async fn main() {
    let state = Arc::new(AppState::new());
    let app = create_router(state);
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### MCP Server Implementation

```rust
use mcp_sdk::{Server, Tool, Resource};

pub struct CuraMcpServer {
    cura_client: CuraApiClient,
}

impl CuraMcpServer {
    pub async fn run(&self, transport: impl Transport) -> Result<()> {
        let server = Server::new("cura", "1.0.0")
            .with_tool(Tool::new("load_model")
                .description("Load a 3D model into the scene")
                .handler(|params| self.handle_load_model(params)))
            .with_tool(Tool::new("slice")
                .description("Slice the scene to generate G-code")
                .handler(|params| self.handle_slice(params)))
            .with_tool(Tool::new("set_machine")
                .description("Set the active printer")
                .handler(|params| self.handle_set_machine(params)))
            // ... more tools
            .with_resource(Resource::new("cura://scene")
                .handler(|| self.get_scene_resource()))
            .with_resource(Resource::new("cura://settings")
                .handler(|| self.get_settings_resource()));
            
        server.serve(transport).await
    }
    
    async fn handle_load_model(&self, params: Value) -> Result<Value> {
        let file_path = params["file_path"].as_str()
            .ok_or_else(|| anyhow\!("file_path is required"))?;
        let auto_arrange = params["auto_arrange"].as_bool().unwrap_or(true);
        
        let model = self.cura_client.add_model(file_path, auto_arrange).await?;
        
        Ok(json\!({
            "success": true,
            "model_id": model.id,
            "name": model.name,
            "bounding_box": model.bounding_box
        }))
    }
    
    async fn handle_slice(&self, params: Value) -> Result<Value> {
        let output_path = params["output_path"].as_str()
            .ok_or_else(|| anyhow\!("output_path is required"))?;
        
        let job = self.cura_client.start_slice().await?;
        
        // Wait for completion
        loop {
            let status = self.cura_client.get_slice_status(&job.id).await?;
            if status.status == "completed" {
                // Download and save G-code
                let gcode = self.cura_client.download_gcode(&job.id).await?;
                tokio::fs::write(output_path, &gcode).await?;
                
                return Ok(json\!({
                    "success": true,
                    "output_path": output_path,
                    "print_time": status.result.print_time,
                    "material_usage": status.result.material_usage
                }));
            }
            tokio::time::sleep(Duration::from_millis(500)).await;
        }
    }
}
```

---

*Document Version: 1.0*
*Last Updated: 2026-02-05*

