# CLAUDE.md - AI Assistant Guide for Ultimaker Cura

This document provides essential information for AI assistants working with the Ultimaker Cura codebase.

## Project Overview

Ultimaker Cura is an open-source 3D printing slicer application that converts 3D models into G-code instructions for 3D printers. It's built with Python, PyQt6, and QML, using the Uranium framework as its foundation.

**Key Technologies:**
- Python 3.x with PyQt6
- QML for UI components
- Conan 2.7+ for package management
- Uranium framework (3D engine, settings system)
- CuraEngine (C++ slicing backend via libArcus)

## Codebase Structure

```
ultimaker-cura/
├── cura/                   # Main Python source code
│   ├── API/                # Public plugin API
│   ├── Arranging/          # Object placement algorithms
│   ├── Backups/            # Profile backup/restore
│   ├── Machines/           # Machine-specific handling
│   ├── OAuth2/             # Authentication system
│   ├── Operations/         # Undoable scene operations
│   ├── PrinterOutput/      # Device communication
│   ├── Scene/              # Scene graph with decorators
│   ├── Settings/           # Settings management system
│   ├── TaskManagement/     # Background job handling
│   ├── UI/                 # UI management components
│   ├── UltimakerCloud/     # Cloud integration
│   ├── Utils/              # Threading, networking utilities
│   └── CuraApplication.py  # Main application class
├── plugins/                # ~46 bundled plugins
├── resources/              # Themes, definitions, translations
│   ├── definitions/        # Machine definitions (.def.json)
│   ├── extruders/          # Extruder definitions
│   ├── variants/           # Hardware variants
│   ├── quality/            # Quality profiles (.inst.cfg)
│   ├── intent/             # Print intent profiles
│   ├── i18n/               # Translations (19 languages)
│   ├── qml/                # QML UI components
│   ├── themes/             # UI themes
│   └── meshes/             # 3D models for platforms
├── tests/                  # pytest test suite
├── scripts/                # Utility scripts
├── printer-linter/         # Definition file validation
└── packaging/              # Platform-specific packaging
```

## Build System

### Conan Package Manager (Primary)
```bash
# Install Conan configuration
conan config install https://github.com/ultimaker/conan-config.git

# Build options in conanfile.py:
# - enterprise: Enable enterprise features
# - staging: Use staging API endpoints
# - cura_debug_mode: Enable debug logging
# - i18n_extract: Extract translation strings
```

### Running Tests
```bash
# Run all tests
pytest tests/

# Run specific test file
pytest tests/Settings/TestCuraStackBuilder.py

# Run with verbose output
pytest -v tests/
```

## Code Style Conventions

### Python Style (from .pylintrc)

**Naming Conventions:**
- Methods: `camelCase` starting with lowercase - `getSomething()`, `_privateSomething()`
- Private methods: `_methodName()` or `__methodName()`
- Magic methods: `__init__()`, `__repr__()`
- Classes: `PascalCase`
- Constants: `UPPER_CASE`

**Formatting:**
- Maximum line length: **120 characters**
- Maximum module length: 500 lines
- Use **double quotes** for strings
- Max 7 arguments per function
- Max 12 branches per function
- Max 5 nested blocks

**Example Method Signature:**
```python
def getSomething(self, parameter: str, optional: Optional[int] = None) -> Dict[str, Any]:
    """Brief description of what this does."""
    pass
```

### Type Hints
Always use type hints with imports from `typing`:
```python
from typing import Optional, Dict, List, Set, TYPE_CHECKING, cast

if TYPE_CHECKING:
    from cura.Settings.MachineManager import MachineManager
```

### PyQt Patterns
```python
from PyQt6.QtCore import pyqtSignal, pyqtProperty, pyqtSlot, QObject

class MyClass(QObject):
    # Signal definition
    mySignal = pyqtSignal()
    settingChanged = pyqtSignal(str)

    # Property with notification
    @pyqtProperty(str, notify=settingChanged)
    def value(self) -> str:
        return self._value

    # Slot for QML/signal connections
    @pyqtSlot(str)
    def onValueChanged(self, new_value: str) -> None:
        self._value = new_value
```

### Copyright Headers
All source files must include:
```python
# Copyright (c) 2024 UltiMaker
# Cura is released under the terms of the LGPLv3 or higher.
```

## Key Architectural Patterns

### 1. Decorator Pattern (Scene Nodes)
```python
from UM.Scene.SceneNodeDecorator import SceneNodeDecorator

class MyDecorator(SceneNodeDecorator):
    def __init__(self) -> None:
        super().__init__()

    def getMyProperty(self) -> str:
        return self._my_property
```

### 2. Container Stack Pattern (Settings)
Settings cascade through container stacks:
```
Definition → Definition Changes → Quality → Quality Changes → Material → Variant → Intent → User
```

### 3. Signal-Driven Communication
Use Qt signals for loose coupling between components:
```python
# Emit
self.mySignal.emit()

# Connect
some_object.mySignal.connect(self._onMySignal)
```

### 4. Singleton Pattern
```python
class MySingleton:
    __instance = None

    @classmethod
    def getInstance(cls) -> "MySingleton":
        if cls.__instance is None:
            cls.__instance = MySingleton()
        return cls.__instance
```

### 5. Job Pattern (Background Tasks)
```python
from UM.Job import Job

class MyJob(Job):
    def __init__(self) -> None:
        super().__init__()

    def run(self) -> None:
        # Long-running operation
        self.setResult(result)
```

### 6. Operation Pattern (Undo/Redo)
```python
from UM.Operations.Operation import Operation

class MyOperation(Operation):
    def undo(self) -> None:
        pass

    def redo(self) -> None:
        pass
```

## Plugin Development

### Plugin Structure
```
plugins/MyPlugin/
├── __init__.py
├── plugin.json          # Metadata
├── MyPlugin.py          # Main plugin class
└── qml/                 # Optional QML UI
    └── MyPluginPanel.qml
```

### plugin.json Format
```json
{
    "name": "My Plugin",
    "author": "Author Name",
    "version": "1.0.0",
    "api": 8,
    "description": "Plugin description",
    "i18n-catalog": "cura"
}
```

### Plugin Categories
- **File I/O**: Readers/Writers (3MF, G-code, STL)
- **Stages**: UI workflow stages (Prepare, Preview, Monitor)
- **Views**: Rendering modes (Solid, X-Ray, Simulation)
- **Tools**: Interactive scene tools (Support Eraser, Paint)
- **Output Devices**: Printer connectivity (USB, Network)
- **Post-Processing**: G-code modification scripts
- **Version Upgrades**: Configuration migration

## Testing Guidelines

### Test File Location
- Module tests: `tests/` directory
- Plugin tests: Within plugin directory

### pytest Configuration (pytest.ini)
```ini
[pytest]
testpaths = tests
python_files = Test*.py
python_classes = Test
```

### Test Patterns
```python
import pytest
from unittest.mock import MagicMock, patch

class TestMyFeature:
    @pytest.fixture
    def mock_application(self):
        return MagicMock()

    def test_something(self, mock_application):
        # Test implementation
        assert result == expected
```

### Common Mocks
- `CuraApplication` - Main application
- `ContainerRegistry` - Settings container storage
- `MachineManager` - Machine configuration
- `ExtruderManager` - Extruder settings

## Resource Files

### Machine Definitions (.def.json)
```json
{
    "version": 2,
    "name": "My Printer",
    "inherits": "fdmprinter",
    "metadata": {
        "manufacturer": "Manufacturer",
        "visible": true,
        "platform": "my_printer_platform.obj"
    },
    "overrides": {
        "machine_width": { "default_value": 200 },
        "machine_depth": { "default_value": 200 }
    }
}
```

### Quality Profiles (.inst.cfg)
```ini
[general]
version = 4
name = Normal
definition = my_printer

[metadata]
setting_version = 23
type = quality
quality_type = normal

[values]
layer_height = 0.2
```

## CI/CD Workflows

### Main Workflows
- `unit-test.yml` - Runs pytest on PR/push
- `conan-package.yml` - Builds Conan packages
- `printer-linter-*.yml` - Validates definition files
- `linux.yml`, `macos.yml`, `windows.yml` - Platform builds

### Triggering Tests
Tests run automatically when modifying:
- `cura/**`
- `plugins/**`
- `resources/**`
- `tests/**`

## Common Tasks

### Adding a New Setting
1. Add to definition in `resources/definitions/fdmprinter.def.json`
2. Implement handling in appropriate Settings module
3. Add translations to `resources/i18n/`
4. Write tests in `tests/Settings/`

### Adding a New Printer
1. Create definition file in `resources/definitions/`
2. Add extruder definitions in `resources/extruders/`
3. Add quality profiles in `resources/quality/`
4. Run printer-linter for validation

### Modifying UI
1. QML files in `resources/qml/`
2. Connect to Python via signals/properties
3. Theme values in `resources/themes/`

## Important Notes for AI Assistants

### DO:
- Read existing code before making modifications
- Follow the established camelCase naming convention
- Use type hints for all function signatures
- Keep changes minimal and focused
- Use signals for component communication
- Write tests for new functionality
- Include copyright headers in new files

### DON'T:
- Use single quotes for strings (use double quotes)
- Create unnecessary abstractions
- Skip type hints
- Modify multiple unrelated systems in one change
- Ignore the container stack hierarchy for settings
- Bypass the signal system for direct method calls

### Key Files to Understand
- `cura/CuraApplication.py` - Application entry point
- `cura/Settings/MachineManager.py` - Machine configuration hub
- `cura/Settings/CuraContainerStack.py` - Settings hierarchy
- `cura/Scene/CuraSceneNode.py` - Scene object handling

### Related Repositories
- [Uranium](https://github.com/Ultimaker/Uranium) - Core framework
- [CuraEngine](https://github.com/Ultimaker/CuraEngine) - Slicing backend
- [fdm_materials](https://github.com/Ultimaker/fdm_materials) - Material database
- [libArcus](https://github.com/Ultimaker/libArcus) - Communication library
- [libSavitar](https://github.com/Ultimaker/libSavitar) - 3MF support
- [libCharon](https://github.com/Ultimaker/libCharon) - Material definitions

## Quick Reference

| Task | Command |
|------|---------|
| Run tests | `pytest tests/` |
| Run specific test | `pytest tests/path/TestFile.py::test_name` |
| Install Conan config | `conan config install https://github.com/ultimaker/conan-config.git` |
| Lint code | `pylint cura/ --rcfile=.pylintrc` |
| Validate definitions | Run printer-linter via CI |
