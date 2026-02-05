# Python → Rust Dependency Mapping

This document maps Cura's Python dependencies to their Rust equivalents, focusing on core functionality needed for the slicer.

## Dependency Categories

| Category | Port Priority | Notes |
|----------|--------------|-------|
| **Core Logic** | P0 | Essential for slicing functionality |
| **File I/O** | P0 | Required for model/profile loading |
| **Geometry** | P0 | Mesh processing, collision detection |
| **Networking** | P1 | Printer discovery, device communication |
| **Cloud** | P2 | Can be deferred or reimplemented |
| **UI** | N/A | Replaced by Tauri frontend |
| **Monitoring** | Skip | Not essential for core functionality |

---

## Core Logic Dependencies

### 1. Uranium Framework

**What it provides:**
- Scene graph and 3D rendering
- Settings management (container stacks)
- Plugin system
- Signal/event system
- File I/O abstractions

**Rust Strategy:** Custom implementation across multiple crates

| Uranium Component | Rust Implementation |
|-------------------|---------------------|
| `UM.Scene.SceneNode` | `cura-scene` crate with custom scene graph |
| `UM.Settings.ContainerStack` | `cura-settings` crate |
| `UM.Settings.DefinitionContainer` | Serde-based JSON parser |
| `UM.Settings.InstanceContainer` | Custom INI-like parser |
| `UM.Signal.Signal` | `tokio::sync::broadcast` channels |
| `UM.PluginRegistry` | Custom plugin trait + dynamic loading |
| `UM.Job` | `tokio` tasks with progress channels |
| `UM.Application` | No equivalent needed (API-first) |

```toml
# cura-scene/Cargo.toml
[dependencies]
nalgebra = "0.32"           # Transform matrices, vectors
parry3d = "0.13"            # Collision detection, convex hulls
uuid = { version = "1", features = ["v4"] }
serde = { version = "1", features = ["derive"] }
```

### 2. CuraEngine Integration

**What it provides:**
- Slicing backend (C++ process)
- Protobuf communication via libArcus

**Rust Strategy:** Reimplement Protobuf client, keep CuraEngine unchanged

| Python Component | Rust Implementation |
|------------------|---------------------|
| `libArcus` (socket comm) | `tonic` gRPC client or raw TCP + `prost` |
| `Cura.proto` messages | Generated with `prost-build` |
| `StartSliceJob` | Async task building protobuf messages |
| `ProcessSlicedLayersJob` | Stream processor for layer data |

```toml
# cura-engine-bridge/Cargo.toml
[dependencies]
prost = "0.12"
tonic = "0.10"
tokio = { version = "1", features = ["net", "process", "io-util"] }
bytes = "1"

[build-dependencies]
prost-build = "0.12"
```

```rust
// build.rs
fn main() {
    prost_build::compile_protos(&["proto/Cura.proto"], &["proto/"])
        .expect("Failed to compile protobuf definitions");
}
```

### 3. NumPy

**What it provides:**
- N-dimensional arrays
- Mathematical operations on mesh data
- Matrix computations

**Rust Equivalent:** `ndarray` + `nalgebra`

```toml
[dependencies]
ndarray = "0.15"            # N-dimensional arrays
nalgebra = "0.32"           # Linear algebra, matrices
num-traits = "0.2"          # Numeric traits
```

**Usage Example:**
```rust
use nalgebra::{Matrix4, Point3, Vector3};
use ndarray::Array2;

// Transform a mesh vertex
let transform: Matrix4<f64> = /* ... */;
let vertex = Point3::new(1.0, 2.0, 3.0);
let transformed = transform.transform_point(&vertex);

// Mesh data as ndarray
let vertices: Array2<f64> = Array2::from_shape_vec(
    (num_vertices, 3),
    vertex_data
)?;
```

---

## Geometry Dependencies

### 4. Trimesh

**What it provides:**
- Mesh loading (STL, OBJ, etc.)
- Mesh manipulation
- Geometry validation

**Rust Equivalent:** `tobj` + `parry3d` + custom utilities

```toml
[dependencies]
tobj = "4"                  # OBJ/MTL loader
parry3d = "0.13"            # Collision, convex hull, mesh operations
stl_io = "0.7"              # STL file I/O
gltf = "1"                  # GLTF/GLB support
```

**Usage Example:**
```rust
use parry3d::shape::TriMesh;
use stl_io::read_stl;

// Load STL file
let file = File::open("model.stl")?;
let stl = read_stl(&mut BufReader::new(file))?;

// Convert to parry3d mesh
let vertices: Vec<Point3<f32>> = stl.vertices.iter()
    .map(|v| Point3::new(v[0], v[1], v[2]))
    .collect();

let indices: Vec<[u32; 3]> = stl.faces.iter()
    .map(|f| [f.vertices[0] as u32, f.vertices[1] as u32, f.vertices[2] as u32])
    .collect();

let mesh = TriMesh::new(vertices, indices);
```

### 5. Shapely

**What it provides:**
- 2D polygon operations
- Spatial analysis
- Boolean operations (union, intersection)

**Rust Equivalent:** `geo` + `geo-types`

```toml
[dependencies]
geo = "0.26"                # Geometric algorithms
geo-types = "0.7"           # Core geometry types
rstar = "0.11"              # Spatial indexing (R-tree)
```

**Usage Example:**
```rust
use geo::{Polygon, BooleanOps, ConvexHull};
use geo_types::{Coord, LineString};

// Create polygon
let exterior = LineString::from(vec![
    Coord { x: 0.0, y: 0.0 },
    Coord { x: 4.0, y: 0.0 },
    Coord { x: 4.0, y: 4.0 },
    Coord { x: 0.0, y: 4.0 },
    Coord { x: 0.0, y: 0.0 },
]);
let polygon = Polygon::new(exterior, vec![]);

// Boolean operations (for support generation)
let union = polygon1.union(&polygon2);
let intersection = polygon1.intersection(&polygon2);

// Convex hull (for build plate arrangement)
let hull = points.convex_hull();
```

### 6. PyNest2D

**What it provides:**
- 2D bin packing / nesting
- Part placement on build plate

**Rust Strategy:** Port algorithm or use `nesting` crate

```toml
[dependencies]
# Option 1: Use existing crate (limited features)
# nesting = "0.1"

# Option 2: Custom implementation using geo primitives
geo = "0.26"
```

**Custom Implementation:**
```rust
pub struct NestingProblem {
    pub bin_width: f64,
    pub bin_height: f64,
    pub items: Vec<NestItem>,
}

pub struct NestItem {
    pub id: usize,
    pub polygon: Polygon<f64>,
    pub rotations: Vec<f64>,  // Allowed rotation angles
}

pub struct NestResult {
    pub placements: Vec<Placement>,
    pub bins_used: usize,
}

pub struct Placement {
    pub item_id: usize,
    pub position: Coord<f64>,
    pub rotation: f64,
    pub bin_index: usize,
}

impl NestingProblem {
    pub fn solve(&self, config: &NestingConfig) -> NestResult {
        // Implement NFP (No-Fit Polygon) based nesting
        // or simpler bottom-left algorithm
    }
}
```

---

## File I/O Dependencies

### 7. pySavitar (3MF Support)

**What it provides:**
- 3MF file parsing (ZIP container + XML)
- Model + metadata extraction

**Rust Equivalent:** `zip` + `quick-xml`

```toml
[dependencies]
zip = "0.6"                 # ZIP archive handling
quick-xml = { version = "0.31", features = ["serialize"] }
serde = { version = "1", features = ["derive"] }
```

**Implementation:**
```rust
use quick_xml::de::from_str;
use zip::ZipArchive;

#[derive(Debug, Deserialize)]
#[serde(rename = "model")]
pub struct ThreeMFModel {
    #[serde(rename = "@unit")]
    pub unit: String,

    pub resources: Resources,
    pub build: Build,
}

#[derive(Debug, Deserialize)]
pub struct Resources {
    #[serde(rename = "object", default)]
    pub objects: Vec<Object>,

    #[serde(rename = "basematerials", default)]
    pub base_materials: Vec<BaseMaterials>,
}

pub fn read_3mf(path: &Path) -> Result<ThreeMFModel> {
    let file = File::open(path)?;
    let mut archive = ZipArchive::new(file)?;

    // Read relationships to find root model
    let rels = read_relationships(&mut archive)?;
    let model_path = find_root_model(&rels)?;

    // Parse model XML
    let mut model_file = archive.by_name(&model_path)?;
    let mut xml = String::new();
    model_file.read_to_string(&mut xml)?;

    let model: ThreeMFModel = from_str(&xml)?;
    Ok(model)
}
```

### 8. G-code Parsing

**What it provides:**
- G-code file reading
- Layer visualization data extraction
- Print time estimation

**Rust Equivalent:** Custom parser

```rust
pub struct GCodeParser {
    flavor: GCodeFlavor,
}

#[derive(Debug, Clone)]
pub enum GCodeFlavor {
    Marlin,
    RepRap,
    Griffin,
    Ultimaker,
}

#[derive(Debug)]
pub struct GCodeCommand {
    pub line_number: usize,
    pub command: char,
    pub code: u32,
    pub params: HashMap<char, f64>,
    pub comment: Option<String>,
}

#[derive(Debug)]
pub struct ParsedGCode {
    pub commands: Vec<GCodeCommand>,
    pub layers: Vec<LayerInfo>,
    pub print_time: Duration,
    pub material_usage: HashMap<usize, f64>,  // extruder -> mm³
}

impl GCodeParser {
    pub fn parse(&self, reader: impl BufRead) -> Result<ParsedGCode> {
        let mut commands = Vec::new();
        let mut layers = Vec::new();
        let mut current_layer = LayerInfo::default();
        let mut current_z = 0.0;

        for (line_num, line) in reader.lines().enumerate() {
            let line = line?;
            let trimmed = line.trim();

            if trimmed.is_empty() || trimmed.starts_with(';') {
                // Check for layer comments
                if let Some(layer_z) = self.parse_layer_comment(trimmed) {
                    if layer_z > current_z {
                        layers.push(std::mem::take(&mut current_layer));
                        current_z = layer_z;
                    }
                }
                continue;
            }

            if let Some(cmd) = self.parse_command(trimmed, line_num) {
                commands.push(cmd);
            }
        }

        Ok(ParsedGCode {
            commands,
            layers,
            print_time: self.estimate_print_time(&commands),
            material_usage: self.calculate_material_usage(&commands),
        })
    }
}
```

### 9. Profile Parsing (YAML/INI)

**What it provides:**
- Quality profile reading (.inst.cfg)
- Machine definition parsing (.def.json)
- Material profile handling

**Rust Equivalent:** `serde_yaml` + `serde_json` + custom INI

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"
configparser = "3"          # INI-like format
```

**Definition File Parser:**
```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct MachineDefinition {
    pub version: u32,
    pub name: String,
    pub inherits: Option<String>,
    pub metadata: MachineMetadata,
    pub overrides: HashMap<String, SettingOverride>,
    pub settings: Option<HashMap<String, SettingDefinition>>,
}

#[derive(Debug, Deserialize)]
pub struct MachineMetadata {
    pub visible: bool,
    pub author: Option<String>,
    pub manufacturer: Option<String>,
    pub platform: Option<String>,
    pub platform_offset: Option<[f64; 3]>,
    pub machine_extruder_trains: Option<HashMap<String, String>>,
}

#[derive(Debug, Deserialize)]
pub struct SettingDefinition {
    pub label: String,
    pub description: String,
    #[serde(rename = "type")]
    pub value_type: String,
    pub default_value: serde_json::Value,
    pub unit: Option<String>,
    pub minimum_value: Option<f64>,
    pub maximum_value: Option<f64>,
    pub settable_per_mesh: Option<bool>,
    pub settable_per_extruder: Option<bool>,
    pub enabled: Option<String>,
    pub value: Option<String>,
}

pub fn load_definition(path: &Path) -> Result<MachineDefinition> {
    let content = fs::read_to_string(path)?;
    let def: MachineDefinition = serde_json::from_str(&content)?;
    Ok(def)
}
```

**Instance Container (INI) Parser:**
```rust
use configparser::ini::Ini;

#[derive(Debug)]
pub struct InstanceContainer {
    pub id: String,
    pub name: String,
    pub version: u32,
    pub definition: String,
    pub metadata: InstanceMetadata,
    pub values: HashMap<String, String>,
}

#[derive(Debug)]
pub struct InstanceMetadata {
    pub container_type: String,
    pub quality_type: Option<String>,
    pub setting_version: u32,
    pub intent_category: Option<String>,
}

pub fn load_instance_container(path: &Path) -> Result<InstanceContainer> {
    let mut config = Ini::new();
    config.load(path)?;

    let general = config.get_map_ref().get("general")
        .ok_or_else(|| anyhow!("Missing [general] section"))?;

    let metadata_section = config.get_map_ref().get("metadata")
        .ok_or_else(|| anyhow!("Missing [metadata] section"))?;

    let values = config.get_map_ref().get("values")
        .cloned()
        .unwrap_or_default();

    Ok(InstanceContainer {
        id: path.file_stem()
            .and_then(|s| s.to_str())
            .unwrap_or_default()
            .to_string(),
        name: general.get("name")
            .and_then(|v| v.clone())
            .unwrap_or_default(),
        version: general.get("version")
            .and_then(|v| v.as_ref().and_then(|s| s.parse().ok()))
            .unwrap_or(4),
        definition: general.get("definition")
            .and_then(|v| v.clone())
            .unwrap_or_default(),
        metadata: parse_metadata(metadata_section)?,
        values: values.into_iter()
            .filter_map(|(k, v)| v.map(|val| (k, val)))
            .collect(),
    })
}
```

---

## Networking Dependencies

### 10. Requests (HTTP Client)

**What it provides:**
- HTTP requests for cloud API
- Material library downloads
- Firmware updates

**Rust Equivalent:** `reqwest`

```toml
[dependencies]
reqwest = { version = "0.11", features = ["json", "multipart", "stream"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

**Usage:**
```rust
use reqwest::Client;

pub struct CloudClient {
    client: Client,
    base_url: String,
    auth_token: Option<String>,
}

impl CloudClient {
    pub async fn get_materials(&self) -> Result<Vec<Material>> {
        let response = self.client
            .get(format!("{}/materials", self.base_url))
            .bearer_auth(self.auth_token.as_deref().unwrap_or_default())
            .send()
            .await?
            .error_for_status()?
            .json::<Vec<Material>>()
            .await?;

        Ok(response)
    }
}
```

### 11. Zeroconf (mDNS/Bonjour)

**What it provides:**
- Local network printer discovery
- Service announcement

**Rust Equivalent:** `mdns-sd`

```toml
[dependencies]
mdns-sd = "0.10"
```

**Usage:**
```rust
use mdns_sd::{ServiceDaemon, ServiceEvent};

pub struct PrinterDiscovery {
    daemon: ServiceDaemon,
}

impl PrinterDiscovery {
    pub fn new() -> Result<Self> {
        let daemon = ServiceDaemon::new()?;
        Ok(Self { daemon })
    }

    pub async fn discover_printers(&self) -> Result<Vec<DiscoveredPrinter>> {
        let receiver = self.daemon.browse("_ultimaker._tcp.local.")?;
        let mut printers = Vec::new();

        // Listen for a few seconds
        let timeout = tokio::time::sleep(Duration::from_secs(5));
        tokio::pin!(timeout);

        loop {
            tokio::select! {
                event = receiver.recv_async() => {
                    match event {
                        Ok(ServiceEvent::ServiceResolved(info)) => {
                            printers.push(DiscoveredPrinter {
                                name: info.get_fullname().to_string(),
                                ip: info.get_addresses().iter().next().copied(),
                                port: info.get_port(),
                                properties: info.get_properties().clone(),
                            });
                        }
                        _ => {}
                    }
                }
                _ = &mut timeout => break,
            }
        }

        Ok(printers)
    }
}
```

### 12. Keyring (Credential Storage)

**What it provides:**
- Secure token storage
- OS keychain integration

**Rust Equivalent:** `keyring`

```toml
[dependencies]
keyring = "2"
```

**Usage:**
```rust
use keyring::Entry;

const SERVICE_NAME: &str = "cura-rs";

pub struct CredentialStore;

impl CredentialStore {
    pub fn store_token(user: &str, token: &str) -> Result<()> {
        let entry = Entry::new(SERVICE_NAME, user)?;
        entry.set_password(token)?;
        Ok(())
    }

    pub fn get_token(user: &str) -> Result<Option<String>> {
        let entry = Entry::new(SERVICE_NAME, user)?;
        match entry.get_password() {
            Ok(password) => Ok(Some(password)),
            Err(keyring::Error::NoEntry) => Ok(None),
            Err(e) => Err(e.into()),
        }
    }

    pub fn delete_token(user: &str) -> Result<()> {
        let entry = Entry::new(SERVICE_NAME, user)?;
        entry.delete_password()?;
        Ok(())
    }
}
```

### 13. Serial Communication

**What it provides:**
- USB printer connectivity
- Serial port communication

**Rust Equivalent:** `serialport`

```toml
[dependencies]
serialport = "4"
```

**Usage:**
```rust
use serialport::{SerialPort, available_ports, SerialPortType};
use std::time::Duration;

pub struct UsbPrinter {
    port: Box<dyn SerialPort>,
}

impl UsbPrinter {
    pub fn discover() -> Result<Vec<UsbPrinterInfo>> {
        let ports = available_ports()?;
        let mut printers = Vec::new();

        for port in ports {
            if let SerialPortType::UsbPort(usb_info) = port.port_type {
                // Check for known printer VID/PID
                if is_known_printer(&usb_info) {
                    printers.push(UsbPrinterInfo {
                        port_name: port.port_name,
                        manufacturer: usb_info.manufacturer.unwrap_or_default(),
                        product: usb_info.product.unwrap_or_default(),
                        serial_number: usb_info.serial_number,
                    });
                }
            }
        }

        Ok(printers)
    }

    pub fn connect(port_name: &str) -> Result<Self> {
        let port = serialport::new(port_name, 115200)
            .timeout(Duration::from_millis(100))
            .open()?;

        Ok(Self { port })
    }

    pub fn send_gcode(&mut self, command: &str) -> Result<String> {
        // Send command
        write!(self.port, "{}\n", command)?;

        // Read response
        let mut response = String::new();
        let mut buf = [0u8; 256];

        loop {
            match self.port.read(&mut buf) {
                Ok(n) if n > 0 => {
                    response.push_str(&String::from_utf8_lossy(&buf[..n]));
                    if response.contains("ok") || response.contains("error") {
                        break;
                    }
                }
                _ => break,
            }
        }

        Ok(response)
    }
}
```

---

## Async Runtime

### 14. Twisted → Tokio

**What it provides:**
- Event loop
- Async networking
- Background tasks

**Rust Equivalent:** `tokio`

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
async-trait = "0.1"
futures = "0.3"
```

**Job Pattern Example:**
```rust
use tokio::sync::{mpsc, oneshot};

pub struct JobManager {
    sender: mpsc::Sender<JobRequest>,
}

pub struct JobRequest {
    job: Box<dyn Job>,
    progress_tx: mpsc::Sender<Progress>,
    result_tx: oneshot::Sender<JobResult>,
}

#[async_trait::async_trait]
pub trait Job: Send + Sync {
    async fn run(&self, progress: mpsc::Sender<Progress>) -> JobResult;
    fn name(&self) -> &str;
}

impl JobManager {
    pub fn new() -> Self {
        let (sender, mut receiver) = mpsc::channel::<JobRequest>(32);

        tokio::spawn(async move {
            while let Some(request) = receiver.recv().await {
                let result = request.job.run(request.progress_tx).await;
                let _ = request.result_tx.send(result);
            }
        });

        Self { sender }
    }

    pub async fn submit<J: Job + 'static>(
        &self,
        job: J,
    ) -> (mpsc::Receiver<Progress>, oneshot::Receiver<JobResult>) {
        let (progress_tx, progress_rx) = mpsc::channel(16);
        let (result_tx, result_rx) = oneshot::channel();

        self.sender.send(JobRequest {
            job: Box::new(job),
            progress_tx,
            result_tx,
        }).await.expect("Job manager channel closed");

        (progress_rx, result_rx)
    }
}
```

---

## Dependencies NOT Ported

### PyQt6 / QML
**Reason:** Replaced by Tauri + web frontend
**Alternative:** Tauri IPC for desktop, REST API for web

### Sentry SDK
**Reason:** Not essential for core functionality
**Alternative:** Structured logging with `tracing`

### 3DConnexion (pynavlib)
**Reason:** Niche hardware support
**Alternative:** Could be added later via native Tauri plugin

---

## Complete Cargo.toml

```toml
# Workspace root Cargo.toml

[workspace]
resolver = "2"
members = [
    "crates/cura-core",
    "crates/cura-settings",
    "crates/cura-scene",
    "crates/cura-machines",
    "crates/cura-engine-bridge",
    "crates/cura-formats",
    "crates/cura-devices",
    "crates/cura-api",
]

[workspace.dependencies]
# Async runtime
tokio = { version = "1", features = ["full"] }
async-trait = "0.1"
futures = "0.3"

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"

# Geometry and math
nalgebra = "0.32"
parry3d = "0.13"
geo = "0.26"
ndarray = "0.15"

# Mesh handling
tobj = "4"
stl_io = "0.7"
gltf = "1"

# 3MF format
zip = "0.6"
quick-xml = { version = "0.31", features = ["serialize"] }

# G-code
# (custom implementation)

# Protobuf (CuraEngine communication)
prost = "0.12"
tonic = "0.10"
bytes = "1"

# HTTP client
reqwest = { version = "0.11", features = ["json", "stream"] }

# mDNS discovery
mdns-sd = "0.10"

# Serial communication
serialport = "4"

# Credential storage
keyring = "2"

# Configuration
config = "0.13"
configparser = "3"
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

# UUID
uuid = { version = "1", features = ["v4", "serde"] }

# Date/time
chrono = { version = "0.4", features = ["serde"] }

[workspace.package]
version = "0.1.0"
edition = "2021"
license = "LGPL-3.0-or-later"
repository = "https://github.com/example/cura-rs"
```

---

## Migration Checklist

- [ ] **Core Math**: `numpy` → `ndarray` + `nalgebra`
- [ ] **Scene Graph**: Uranium SceneNode → custom `cura-scene`
- [ ] **Settings**: Uranium ContainerStack → custom `cura-settings`
- [ ] **Geometry 2D**: `shapely` → `geo`
- [ ] **Geometry 3D**: `trimesh` → `parry3d` + `tobj`
- [ ] **3MF**: `pysavitar` → `zip` + `quick-xml`
- [ ] **G-code**: custom parser (both Python and Rust)
- [ ] **Protobuf**: `protobuf` → `prost` + `tonic`
- [ ] **HTTP**: `requests` → `reqwest`
- [ ] **mDNS**: `zeroconf` → `mdns-sd`
- [ ] **Serial**: `pyserial` → `serialport`
- [ ] **Keyring**: `keyring` → `keyring` (Rust)
- [ ] **Async**: `Twisted` → `tokio`
- [ ] **Nesting**: `pynest2d` → custom or `geo`-based

---

*Document Version: 1.0*
*Last Updated: 2026-02-05*
