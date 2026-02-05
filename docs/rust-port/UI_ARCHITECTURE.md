# Cura UI Architecture: Tauri + Bun

## Overview

The UI is designed as a separate project that deeply integrates with the Cura Core API. Using Tauri for the desktop shell and Bun for the web frontend, a single codebase can target:

- **Desktop**: Native app via Tauri (Windows, macOS, Linux)
- **Web**: Browser-based access to remote Cura API server
- **Embedded**: Integration into other applications

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           cura-ui Repository                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │                    Web Application (src/)                               ││
│  │                                                                         ││
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐   ││
│  │  │   React 19     │  │   Three.js     │  │    Zustand State       │   ││
│  │  │   Components   │  │   3D Viewer    │  │    Management          │   ││
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘   ││
│  │                                                                         ││
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐   ││
│  │  │  TailwindCSS   │  │   React Query  │  │    OpenAPI Client      │   ││
│  │  │   Styling      │  │   Data Fetch   │  │    (Generated)         │   ││
│  │  └────────────────┘  └────────────────┘  └────────────────────────┘   ││
│  │                                                                         ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                               │                                             │
│                               │ API Calls                                   │
│                               ▼                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │                    API Adapter Layer                                    ││
│  │                                                                         ││
│  │  ┌─────────────────────────┐  ┌─────────────────────────────────────┐ ││
│  │  │   Tauri IPC Adapter     │  │      HTTP/WebSocket Adapter         │ ││
│  │  │   (Desktop Mode)        │  │      (Web/Remote Mode)              │ ││
│  │  └─────────────────────────┘  └─────────────────────────────────────┘ ││
│  │                                                                         ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                               │                                             │
└───────────────────────────────┼─────────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Tauri Shell  │     │  Cura API       │     │   Remote API    │
│  (Embedded    │     │  Server         │     │   Server        │
│   cura-core)  │     │  (Standalone)   │     │   (Cloud/LAN)   │
└───────────────┘     └─────────────────┘     └─────────────────┘
```

---

## Project Structure

```
cura-ui/
├── src/                          # Web application source
│   ├── main.tsx                  # Entry point
│   ├── App.tsx                   # Root component
│   │
│   ├── components/               # React components
│   │   ├── viewer/               # 3D viewport
│   │   │   ├── SceneViewer.tsx   # Main 3D view
│   │   │   ├── BuildPlate.tsx    # Build plate rendering
│   │   │   ├── ModelMesh.tsx     # Individual model rendering
│   │   │   ├── LayerPreview.tsx  # Sliced layer visualization
│   │   │   └── Controls.tsx      # Orbit/pan/zoom controls
│   │   │
│   │   ├── sidebar/              # Settings sidebar
│   │   │   ├── Sidebar.tsx       # Container
│   │   │   ├── MachineSelector.tsx
│   │   │   ├── MaterialSelector.tsx
│   │   │   ├── QualitySelector.tsx
│   │   │   ├── SettingsPanel.tsx
│   │   │   └── SettingInput.tsx
│   │   │
│   │   ├── toolbar/              # Top toolbar
│   │   │   ├── Toolbar.tsx
│   │   │   ├── FileMenu.tsx
│   │   │   ├── ViewMenu.tsx
│   │   │   └── SliceButton.tsx
│   │   │
│   │   ├── panels/               # Floating panels
│   │   │   ├── PrintInfo.tsx     # Print time/material
│   │   │   ├── ModelList.tsx     # Scene object list
│   │   │   ├── LayerSlider.tsx   # Layer navigation
│   │   │   └── ProgressModal.tsx # Slicing progress
│   │   │
│   │   └── common/               # Shared components
│   │       ├── Button.tsx
│   │       ├── Select.tsx
│   │       ├── Slider.tsx
│   │       ├── Input.tsx
│   │       └── Modal.tsx
│   │
│   ├── hooks/                    # React hooks
│   │   ├── useApi.ts             # API client hook
│   │   ├── useScene.ts           # Scene state hook
│   │   ├── useSettings.ts        # Settings hook
│   │   ├── useMachine.ts         # Machine selection hook
│   │   ├── useSlicing.ts         # Slicing state hook
│   │   └── useEvents.ts          # SSE/WebSocket events
│   │
│   ├── stores/                   # Zustand stores
│   │   ├── sceneStore.ts         # Scene models state
│   │   ├── machineStore.ts       # Machine/material config
│   │   ├── settingsStore.ts      # User settings
│   │   ├── slicingStore.ts       # Slicing job state
│   │   └── uiStore.ts            # UI state (panels, etc.)
│   │
│   ├── api/                      # API client
│   │   ├── client.ts             # API adapter
│   │   ├── types.ts              # TypeScript types
│   │   └── generated/            # OpenAPI generated client
│   │
│   ├── lib/                      # Utilities
│   │   ├── three/                # Three.js helpers
│   │   │   ├── meshLoader.ts     # STL/OBJ/3MF loading
│   │   │   ├── gcodePaths.ts     # G-code path rendering
│   │   │   └── materials.ts      # Three.js materials
│   │   ├── format.ts             # Formatting utilities
│   │   └── validation.ts         # Input validation
│   │
│   └── styles/                   # Global styles
│       └── globals.css           # Tailwind + custom CSS
│
├── src-tauri/                    # Tauri Rust backend
│   ├── Cargo.toml
│   ├── tauri.conf.json           # Tauri configuration
│   ├── src/
│   │   ├── main.rs               # Entry point
│   │   ├── commands.rs           # IPC command handlers
│   │   ├── state.rs              # Application state
│   │   ├── menu.rs               # Native menu
│   │   └── tray.rs               # System tray
│   └── icons/                    # App icons
│
├── public/                       # Static assets
│   ├── favicon.ico
│   └── platform_meshes/          # 3D printer platform models
│
├── package.json
├── bun.lockb
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.js
└── postcss.config.js
```

---

## Technology Stack

### Frontend (Web Layer)

| Technology | Purpose | Version |
|------------|---------|---------|
| **Bun** | JavaScript runtime & bundler | 1.x |
| **React** | UI framework | 19.x |
| **TypeScript** | Type safety | 5.x |
| **Vite** | Build tool (via Bun) | 5.x |
| **Three.js** | 3D rendering | 0.160+ |
| **@react-three/fiber** | React Three.js bindings | 8.x |
| **@react-three/drei** | Three.js helpers | 9.x |
| **Zustand** | State management | 4.x |
| **TanStack Query** | Server state | 5.x |
| **TailwindCSS** | Styling | 3.x |
| **Radix UI** | Accessible components | Latest |

### Desktop Shell (Tauri)

| Technology | Purpose | Version |
|------------|---------|---------|
| **Tauri** | Desktop framework | 2.x |
| **Rust** | Backend language | 1.75+ |
| **cura-core** | Embedded slicer (optional) | 0.1.0 |
| **tauri-plugin-**** | Various Tauri plugins | Latest |

---

## Key Components

### 1. 3D Viewer (Three.js)

```tsx
// src/components/viewer/SceneViewer.tsx
import { Canvas } from "@react-three/fiber";
import { OrbitControls, Grid, Environment } from "@react-three/drei";
import { useSceneStore } from "@/stores/sceneStore";
import { BuildPlate } from "./BuildPlate";
import { ModelMesh } from "./ModelMesh";

export function SceneViewer() {
  const models = useSceneStore((s) => s.models);
  const buildVolume = useSceneStore((s) => s.buildVolume);
  const viewMode = useSceneStore((s) => s.viewMode);

  return (
    <Canvas
      camera={{ position: [300, 300, 300], fov: 50 }}
      gl={{ antialias: true, alpha: false }}
    >
      <color attach="background" args={["#1a1a1a"]} />

      {/* Lighting */}
      <ambientLight intensity={0.4} />
      <directionalLight position={[10, 10, 5]} intensity={0.8} />

      {/* Build plate */}
      <BuildPlate
        width={buildVolume.width}
        depth={buildVolume.depth}
        height={buildVolume.height}
      />

      {/* Models */}
      {models.map((model) => (
        <ModelMesh
          key={model.id}
          model={model}
          viewMode={viewMode}
        />
      ))}

      {/* Controls */}
      <OrbitControls
        enableDamping
        dampingFactor={0.05}
        minDistance={50}
        maxDistance={1000}
      />

      {/* Grid */}
      <Grid
        infiniteGrid
        cellSize={10}
        cellThickness={0.5}
        sectionSize={50}
        fadeDistance={500}
      />
    </Canvas>
  );
}
```

### 2. Settings Panel

```tsx
// src/components/sidebar/SettingsPanel.tsx
import { useSettings } from "@/hooks/useSettings";
import { SettingInput } from "./SettingInput";

const COMMON_SETTINGS = [
  { key: "layer_height", category: "quality" },
  { key: "infill_sparse_density", category: "infill" },
  { key: "support_enable", category: "support" },
  { key: "adhesion_type", category: "platform_adhesion" },
];

export function SettingsPanel() {
  const { settings, updateSetting, isLoading } = useSettings();

  if (isLoading) return <LoadingSpinner />;

  return (
    <div className="space-y-4 p-4">
      <h2 className="text-lg font-semibold">Print Settings</h2>

      {COMMON_SETTINGS.map(({ key }) => {
        const setting = settings[key];
        if (!setting) return null;

        return (
          <SettingInput
            key={key}
            setting={setting}
            onChange={(value) => updateSetting(key, value)}
          />
        );
      })}

      <button
        className="text-sm text-blue-500 hover:underline"
        onClick={() => openAdvancedSettings()}
      >
        Show all settings...
      </button>
    </div>
  );
}

// src/components/sidebar/SettingInput.tsx
interface SettingInputProps {
  setting: Setting;
  onChange: (value: any) => void;
}

export function SettingInput({ setting, onChange }: SettingInputProps) {
  const { label, value, type, unit, minimum_value, maximum_value } = setting;

  switch (type) {
    case "float":
    case "int":
      return (
        <div className="space-y-1">
          <label className="text-sm text-gray-400">{label}</label>
          <div className="flex items-center gap-2">
            <input
              type="number"
              value={value}
              min={minimum_value}
              max={maximum_value}
              step={type === "float" ? 0.01 : 1}
              onChange={(e) => onChange(parseFloat(e.target.value))}
              className="w-full rounded bg-gray-800 px-2 py-1"
            />
            {unit && <span className="text-sm text-gray-500">{unit}</span>}
          </div>
        </div>
      );

    case "bool":
      return (
        <div className="flex items-center justify-between">
          <label className="text-sm text-gray-400">{label}</label>
          <Switch checked={value} onChange={onChange} />
        </div>
      );

    case "enum":
      return (
        <div className="space-y-1">
          <label className="text-sm text-gray-400">{label}</label>
          <Select value={value} onValueChange={onChange}>
            {setting.options.map((opt) => (
              <SelectItem key={opt.value} value={opt.value}>
                {opt.label}
              </SelectItem>
            ))}
          </Select>
        </div>
      );

    default:
      return null;
  }
}
```

### 3. API Adapter

```typescript
// src/api/client.ts
import { invoke } from "@tauri-apps/api/core";

export interface ApiConfig {
  mode: "tauri" | "http";
  baseUrl?: string;
}

class CuraApiClient {
  private config: ApiConfig;
  private httpClient?: typeof fetch;

  constructor(config: ApiConfig) {
    this.config = config;
  }

  private async call<T>(method: string, endpoint: string, body?: unknown): Promise<T> {
    if (this.config.mode === "tauri") {
      // Use Tauri IPC
      return invoke<T>(`api_${endpoint.replace(/\//g, "_")}`, { body });
    } else {
      // Use HTTP
      const response = await fetch(`${this.config.baseUrl}/api/v1${endpoint}`, {
        method,
        headers: { "Content-Type": "application/json" },
        body: body ? JSON.stringify(body) : undefined,
      });

      if (!response.ok) {
        const error = await response.json();
        throw new ApiError(error.error.code, error.error.message);
      }

      const data = await response.json();
      return data.data;
    }
  }

  // Scene
  async getScene(): Promise<Scene> {
    return this.call("GET", "/scene");
  }

  async addModel(file: File | string, autoArrange = true): Promise<Model> {
    if (this.config.mode === "tauri" && typeof file === "string") {
      // Desktop: pass file path directly
      return invoke("add_model", { filePath: file, autoArrange });
    } else {
      // Web: upload file
      const formData = new FormData();
      formData.append("file", file);
      formData.append("auto_arrange", String(autoArrange));

      const response = await fetch(`${this.config.baseUrl}/api/v1/scene/models`, {
        method: "POST",
        body: formData,
      });

      const data = await response.json();
      return data.data;
    }
  }

  async updateModel(id: string, updates: Partial<Model>): Promise<Model> {
    return this.call("PATCH", `/scene/models/${id}`, updates);
  }

  async removeModel(id: string): Promise<void> {
    return this.call("DELETE", `/scene/models/${id}`);
  }

  async arrangeModels(): Promise<void> {
    return this.call("POST", "/scene/arrange");
  }

  // Machines
  async listMachines(): Promise<MachineDefinition[]> {
    return this.call("GET", "/machines");
  }

  async getActiveMachine(): Promise<MachineConfig> {
    return this.call("GET", "/machines/active");
  }

  async setActiveMachine(machineId: string): Promise<void> {
    return this.call("PUT", "/machines/active", { machine_id: machineId });
  }

  // Materials
  async listMaterials(): Promise<Material[]> {
    return this.call("GET", "/materials");
  }

  async setActiveMaterial(extruder: number, materialId: string): Promise<void> {
    return this.call("PUT", "/materials/active", { extruder, material_id: materialId });
  }

  // Settings
  async getSettings(): Promise<Record<string, Setting>> {
    return this.call("GET", "/settings");
  }

  async updateSettings(settings: Record<string, unknown>): Promise<void> {
    return this.call("PATCH", "/settings", settings);
  }

  // Slicing
  async startSlice(): Promise<SliceJob> {
    return this.call("POST", "/slice");
  }

  async getSliceStatus(jobId: string): Promise<SliceJob> {
    return this.call("GET", `/slice/${jobId}`);
  }

  async cancelSlice(jobId: string): Promise<void> {
    return this.call("DELETE", `/slice/${jobId}`);
  }

  async downloadGcode(jobId: string): Promise<Blob> {
    if (this.config.mode === "tauri") {
      const gcode = await invoke<string>("get_gcode", { jobId });
      return new Blob([gcode], { type: "text/plain" });
    } else {
      const response = await fetch(`${this.config.baseUrl}/api/v1/slice/${jobId}/gcode`);
      return response.blob();
    }
  }

  // Events
  subscribeToEvents(onEvent: (event: ServerEvent) => void): () => void {
    if (this.config.mode === "tauri") {
      // Tauri event listener
      const unlisten = listen<ServerEvent>("cura-event", (event) => {
        onEvent(event.payload);
      });
      return () => unlisten.then((fn) => fn());
    } else {
      // SSE
      const eventSource = new EventSource(`${this.config.baseUrl}/api/v1/events`);
      eventSource.onmessage = (event) => {
        onEvent(JSON.parse(event.data));
      };
      return () => eventSource.close();
    }
  }
}

// Create singleton based on environment
export const api = new CuraApiClient({
  mode: window.__TAURI__ ? "tauri" : "http",
  baseUrl: import.meta.env.VITE_API_URL || "http://localhost:8080",
});
```

### 4. Zustand Store

```typescript
// src/stores/sceneStore.ts
import { create } from "zustand";
import { api } from "@/api/client";

interface Model {
  id: string;
  name: string;
  path: string;
  transform: {
    position: [number, number, number];
    rotation: [number, number, number, number];
    scale: [number, number, number];
  };
  boundingBox: {
    min: [number, number, number];
    max: [number, number, number];
  };
  settingsOverride: Record<string, unknown>;
  selected: boolean;
}

interface SceneState {
  models: Model[];
  buildVolume: { width: number; depth: number; height: number };
  viewMode: "solid" | "xray" | "layers";
  selectedModelId: string | null;

  // Actions
  fetchScene: () => Promise<void>;
  addModel: (file: File | string) => Promise<void>;
  removeModel: (id: string) => Promise<void>;
  updateModelTransform: (id: string, transform: Partial<Model["transform"]>) => void;
  selectModel: (id: string | null) => void;
  arrangeModels: () => Promise<void>;
  setViewMode: (mode: SceneState["viewMode"]) => void;
}

export const useSceneStore = create<SceneState>((set, get) => ({
  models: [],
  buildVolume: { width: 235, depth: 235, height: 250 },
  viewMode: "solid",
  selectedModelId: null,

  fetchScene: async () => {
    const scene = await api.getScene();
    set({
      models: scene.models.map((m) => ({ ...m, selected: false })),
      buildVolume: scene.build_plate,
    });
  },

  addModel: async (file) => {
    const model = await api.addModel(file);
    set((state) => ({
      models: [...state.models, { ...model, selected: false }],
    }));
  },

  removeModel: async (id) => {
    await api.removeModel(id);
    set((state) => ({
      models: state.models.filter((m) => m.id !== id),
      selectedModelId: state.selectedModelId === id ? null : state.selectedModelId,
    }));
  },

  updateModelTransform: (id, transform) => {
    set((state) => ({
      models: state.models.map((m) =>
        m.id === id ? { ...m, transform: { ...m.transform, ...transform } } : m
      ),
    }));
    // Debounced sync to API
    debouncedUpdateModel(id, { transform: get().models.find((m) => m.id === id)?.transform });
  },

  selectModel: (id) => {
    set((state) => ({
      selectedModelId: id,
      models: state.models.map((m) => ({ ...m, selected: m.id === id })),
    }));
  },

  arrangeModels: async () => {
    await api.arrangeModels();
    await get().fetchScene();
  },

  setViewMode: (mode) => set({ viewMode: mode }),
}));
```

### 5. Slicing with Progress

```typescript
// src/stores/slicingStore.ts
import { create } from "zustand";
import { api } from "@/api/client";

interface SlicingState {
  isSlicing: boolean;
  progress: number;
  currentLayer: number;
  totalLayers: number;
  result: SliceResult | null;
  error: string | null;

  startSlice: () => Promise<void>;
  cancelSlice: () => void;
  reset: () => void;
}

export const useSlicingStore = create<SlicingState>((set, get) => ({
  isSlicing: false,
  progress: 0,
  currentLayer: 0,
  totalLayers: 0,
  result: null,
  error: null,

  startSlice: async () => {
    set({ isSlicing: true, progress: 0, error: null, result: null });

    try {
      const job = await api.startSlice();

      // Subscribe to progress events
      const unsubscribe = api.subscribeToEvents((event) => {
        if (event.type === "slice_progress" && event.job_id === job.job_id) {
          set({
            progress: event.progress,
            currentLayer: event.current_layer,
            totalLayers: event.total_layers,
          });
        } else if (event.type === "slice_completed" && event.job_id === job.job_id) {
          set({
            isSlicing: false,
            progress: 1,
            result: event.result,
          });
          unsubscribe();
        } else if (event.type === "slice_error" && event.job_id === job.job_id) {
          set({
            isSlicing: false,
            error: event.error,
          });
          unsubscribe();
        }
      });
    } catch (error) {
      set({ isSlicing: false, error: String(error) });
    }
  },

  cancelSlice: async () => {
    // Implementation
  },

  reset: () => {
    set({
      isSlicing: false,
      progress: 0,
      currentLayer: 0,
      totalLayers: 0,
      result: null,
      error: null,
    });
  },
}));

// src/components/panels/ProgressModal.tsx
import { useSlicingStore } from "@/stores/slicingStore";
import { Dialog } from "@radix-ui/react-dialog";

export function SlicingProgressModal() {
  const { isSlicing, progress, currentLayer, totalLayers, result, error } = useSlicingStore();

  if (!isSlicing && !result) return null;

  return (
    <Dialog open={isSlicing || !!result}>
      <DialogContent>
        {isSlicing ? (
          <div className="space-y-4">
            <h2 className="text-lg font-semibold">Slicing...</h2>
            <div className="h-2 w-full rounded-full bg-gray-700">
              <div
                className="h-full rounded-full bg-blue-500 transition-all"
                style={{ width: `${progress * 100}%` }}
              />
            </div>
            <p className="text-sm text-gray-400">
              Layer {currentLayer} of {totalLayers}
            </p>
          </div>
        ) : result ? (
          <div className="space-y-4">
            <h2 className="text-lg font-semibold">Slicing Complete</h2>
            <div className="grid grid-cols-2 gap-4">
              <div>
                <p className="text-sm text-gray-400">Print Time</p>
                <p className="text-xl font-medium">{result.print_time.formatted}</p>
              </div>
              <div>
                <p className="text-sm text-gray-400">Material</p>
                <p className="text-xl font-medium">{result.material_usage[0].weight_g}g</p>
              </div>
            </div>
            <div className="flex gap-2">
              <Button onClick={downloadGcode}>Save G-code</Button>
              <Button variant="outline" onClick={closeModal}>Close</Button>
            </div>
          </div>
        ) : null}
      </DialogContent>
    </Dialog>
  );
}
```

---

## Tauri Backend

### IPC Commands

```rust
// src-tauri/src/commands.rs
use tauri::State;
use cura_core::{Scene, MachineManager, SlicingService};
use std::sync::Arc;
use tokio::sync::RwLock;

pub struct AppState {
    pub scene: Arc<RwLock<Scene>>,
    pub machine_manager: Arc<RwLock<MachineManager>>,
    pub slicing_service: Arc<SlicingService>,
}

#[tauri::command]
pub async fn get_scene(state: State<'_, AppState>) -> Result<SceneDto, String> {
    let scene = state.scene.read().await;
    Ok(scene.to_dto())
}

#[tauri::command]
pub async fn add_model(
    state: State<'_, AppState>,
    file_path: String,
    auto_arrange: bool,
) -> Result<ModelDto, String> {
    let mut scene = state.scene.write().await;
    let model = scene.load_model(&file_path)
        .map_err(|e| e.to_string())?;

    if auto_arrange {
        scene.auto_arrange();
    }

    Ok(model.to_dto())
}

#[tauri::command]
pub async fn start_slice(
    state: State<'_, AppState>,
    app: tauri::AppHandle,
) -> Result<SliceJobDto, String> {
    let scene = state.scene.read().await;
    let machine = state.machine_manager.read().await;

    let job_id = state.slicing_service
        .start_slice(&scene, &machine)
        .map_err(|e| e.to_string())?;

    // Spawn progress listener
    let service = state.slicing_service.clone();
    let handle = app.clone();
    tokio::spawn(async move {
        let mut progress_rx = service.subscribe_progress(&job_id);
        while let Ok(progress) = progress_rx.recv().await {
            handle.emit_all("cura-event", SliceProgressEvent {
                job_id: job_id.clone(),
                progress: progress.progress,
                current_layer: progress.current_layer,
                total_layers: progress.total_layers,
            }).ok();
        }
    });

    Ok(SliceJobDto { job_id, status: "running".to_string() })
}

#[tauri::command]
pub async fn get_settings(state: State<'_, AppState>) -> Result<SettingsDto, String> {
    let machine = state.machine_manager.read().await;
    let settings = machine.get_resolved_settings();
    Ok(settings.to_dto())
}

#[tauri::command]
pub async fn update_settings(
    state: State<'_, AppState>,
    settings: std::collections::HashMap<String, serde_json::Value>,
) -> Result<(), String> {
    let mut machine = state.machine_manager.write().await;
    for (key, value) in settings {
        machine.set_user_setting(&key, value)
            .map_err(|e| e.to_string())?;
    }
    Ok(())
}

// File dialogs
#[tauri::command]
pub async fn open_file_dialog() -> Result<Option<String>, String> {
    use tauri_plugin_dialog::DialogExt;

    let file = tauri::api::dialog::blocking::FileDialogBuilder::new()
        .add_filter("3D Models", &["stl", "obj", "3mf", "amf"])
        .pick_file();

    Ok(file.map(|p| p.to_string_lossy().to_string()))
}

#[tauri::command]
pub async fn save_gcode_dialog(gcode: String) -> Result<Option<String>, String> {
    use tauri_plugin_dialog::DialogExt;

    let file = tauri::api::dialog::blocking::FileDialogBuilder::new()
        .add_filter("G-code", &["gcode", "gco"])
        .save_file();

    if let Some(path) = file {
        std::fs::write(&path, &gcode)
            .map_err(|e| e.to_string())?;
        Ok(Some(path.to_string_lossy().to_string()))
    } else {
        Ok(None)
    }
}
```

### Tauri Configuration

```json
// src-tauri/tauri.conf.json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "Cura",
  "version": "0.1.0",
  "identifier": "com.cura-rs.app",

  "build": {
    "beforeBuildCommand": "bun run build",
    "beforeDevCommand": "bun run dev",
    "devUrl": "http://localhost:5173",
    "frontendDist": "../dist"
  },

  "app": {
    "windows": [
      {
        "title": "Cura",
        "width": 1400,
        "height": 900,
        "resizable": true,
        "fullscreen": false,
        "minWidth": 1000,
        "minHeight": 700
      }
    ],
    "security": {
      "csp": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'"
    }
  },

  "bundle": {
    "active": true,
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ],
    "targets": ["app", "dmg", "msi", "deb"],
    "macOS": {
      "minimumSystemVersion": "10.15"
    }
  },

  "plugins": {
    "dialog": {},
    "fs": {
      "scope": ["$DOCUMENT/*", "$HOME/*"]
    }
  }
}
```

---

## Build & Deployment

### Development

```bash
# Install dependencies
bun install

# Start development (web only)
bun run dev

# Start development (Tauri desktop)
bun run tauri dev
```

### Production Build

```bash
# Build web version
bun run build

# Build desktop (all platforms)
bun run tauri build

# Build for specific platform
bun run tauri build --target x86_64-pc-windows-msvc
bun run tauri build --target x86_64-apple-darwin
bun run tauri build --target x86_64-unknown-linux-gnu
```

### Package.json Scripts

```json
{
  "name": "cura-ui",
  "version": "0.1.0",
  "scripts": {
    "dev": "vite",
    "dev:tauri": "tauri dev",
    "build": "tsc && vite build",
    "build:tauri": "tauri build",
    "preview": "vite preview",
    "lint": "eslint src --ext ts,tsx",
    "typecheck": "tsc --noEmit",
    "generate:api": "openapi-typescript ../cura-api/openapi.yaml -o src/api/generated/types.ts"
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@react-three/fiber": "^8.15.0",
    "@react-three/drei": "^9.88.0",
    "three": "^0.160.0",
    "zustand": "^4.4.0",
    "@tanstack/react-query": "^5.0.0",
    "@radix-ui/react-dialog": "^1.0.0",
    "@radix-ui/react-select": "^2.0.0",
    "@radix-ui/react-slider": "^1.0.0",
    "@radix-ui/react-switch": "^1.0.0",
    "@tauri-apps/api": "^2.0.0",
    "@tauri-apps/plugin-dialog": "^2.0.0",
    "@tauri-apps/plugin-fs": "^2.0.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@types/three": "^0.160.0",
    "@tauri-apps/cli": "^2.0.0",
    "typescript": "^5.3.0",
    "vite": "^5.0.0",
    "@vitejs/plugin-react": "^4.2.0",
    "tailwindcss": "^3.4.0",
    "postcss": "^8.4.0",
    "autoprefixer": "^10.4.0",
    "eslint": "^8.55.0",
    "openapi-typescript": "^6.7.0"
  }
}
```

---

## Responsive Design

The UI adapts to different screen sizes:

```tsx
// src/App.tsx
export function App() {
  const isMobile = useMediaQuery("(max-width: 768px)");

  return (
    <div className="h-screen flex flex-col">
      <Toolbar />

      <div className="flex-1 flex">
        {/* 3D Viewer - always visible */}
        <div className={cn(
          "flex-1",
          isMobile ? "h-[60vh]" : "h-full"
        )}>
          <SceneViewer />
        </div>

        {/* Sidebar - collapsible on mobile */}
        {!isMobile && (
          <div className="w-80 border-l border-gray-700 overflow-y-auto">
            <Sidebar />
          </div>
        )}
      </div>

      {/* Bottom sheet on mobile */}
      {isMobile && <MobileBottomSheet />}

      {/* Modals */}
      <SlicingProgressModal />
      <SettingsModal />
    </div>
  );
}
```

---

## Performance Optimizations

1. **Virtual Lists**: For long settings lists
2. **Mesh LOD**: Level-of-detail for complex models
3. **Web Workers**: Heavy computation off main thread
4. **Lazy Loading**: Split routes for faster initial load
5. **Memoization**: React.memo for expensive components
6. **Instanced Rendering**: For duplicate models

---

*Document Version: 1.0*
*Last Updated: 2026-02-05*
