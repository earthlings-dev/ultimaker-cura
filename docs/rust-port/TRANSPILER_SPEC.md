# py2cura-rs: Programmatic Python→Rust Transpiler Specification

## Overview

`py2cura-rs` is a domain-specific transpiler designed to convert Ultimaker Cura's Python codebase to idiomatic Rust. Unlike general-purpose Python-to-Rust tools, it understands Cura's patterns (Qt signals, container stacks, decorators) and produces clean, maintainable Rust code.

## Design Philosophy

1. **Domain-Specific**: Optimized for Cura patterns, not general Python
2. **Incremental**: Supports partial transpilation and manual overrides
3. **Reproducible**: Same input always produces same output
4. **Diffable**: Easy to track changes between upstream versions
5. **Patchable**: Manual fixes stored as reusable patches

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         py2cura-rs Pipeline                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐ │
│  │  Python  │    │   AST    │    │   IR     │    │    Rust      │ │
│  │  Source  │───▶│  Parser  │───▶│  Build   │───▶│  Generator   │ │
│  │          │    │          │    │          │    │              │ │
│  └──────────┘    └──────────┘    └──────────┘    └──────────────┘ │
│       │               │               │                 │          │
│       │               │               │                 │          │
│       ▼               ▼               ▼                 ▼          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐ │
│  │  Config  │    │   Type   │    │Transform │    │    Patch     │ │
│  │  Files   │    │ Inference│    │  Passes  │    │   Applier    │ │
│  │          │    │          │    │          │    │              │ │
│  └──────────┘    └──────────┘    └──────────┘    └──────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Stage 1: Python AST Parsing

### Parser Selection

Use `rustpython-parser` for native Rust AST parsing:

```rust
use rustpython_parser::{parse, Mode};

pub fn parse_python_file(source: &str, filename: &str) -> Result<ast::Module, ParseError> {
    parse(source, Mode::Module, filename)
        .map_err(|e| ParseError::SyntaxError(e.to_string()))
}
```

### AST Normalization

Normalize Python AST to handle:
- Multiple function signatures (default args, *args, **kwargs)
- Nested classes and functions
- Comprehensions and generators
- Context managers
- Async/await patterns

```rust
pub struct NormalizedAst {
    pub modules: HashMap<String, NormalizedModule>,
}

pub struct NormalizedModule {
    pub path: PathBuf,
    pub imports: Vec<Import>,
    pub classes: Vec<NormalizedClass>,
    pub functions: Vec<NormalizedFunction>,
    pub constants: Vec<NormalizedConstant>,
}

pub struct NormalizedClass {
    pub name: String,
    pub bases: Vec<TypeRef>,
    pub decorators: Vec<Decorator>,
    pub methods: Vec<NormalizedFunction>,
    pub class_vars: Vec<ClassVariable>,
    pub instance_vars: Vec<InstanceVariable>,
    pub signals: Vec<QtSignal>,
    pub properties: Vec<QtProperty>,
}
```

---

## Stage 2: Type Inference

### Type Inference Engine

Since Python is dynamically typed, we need sophisticated type inference:

```rust
pub struct TypeInferenceEngine {
    // Known type annotations from source
    annotations: HashMap<Symbol, PythonType>,
    // Inferred types from usage
    inferred: HashMap<Symbol, PythonType>,
    // Type stubs for external libraries
    stubs: TypeStubRegistry,
    // Cura-specific type knowledge
    cura_types: CuraTypeRegistry,
}

impl TypeInferenceEngine {
    pub fn infer_function(&mut self, func: &NormalizedFunction) -> InferredSignature {
        // 1. Use explicit annotations if present
        // 2. Infer from default values
        // 3. Infer from usage patterns
        // 4. Fall back to Cura-specific heuristics
        // 5. Use Any as last resort (flag for manual review)
    }
}
```

### Type Mapping Rules

```rust
pub enum PythonType {
    // Primitives
    Int,
    Float,
    Bool,
    Str,
    Bytes,
    None,

    // Collections
    List(Box<PythonType>),
    Dict(Box<PythonType>, Box<PythonType>),
    Set(Box<PythonType>),
    Tuple(Vec<PythonType>),
    FrozenSet(Box<PythonType>),

    // Optional/Union
    Optional(Box<PythonType>),
    Union(Vec<PythonType>),

    // Callable
    Callable {
        args: Vec<PythonType>,
        ret: Box<PythonType>,
    },

    // References
    Class(String),
    Generic(String, Vec<PythonType>),

    // Special
    Any,
    TypeVar(String),

    // Cura-specific
    Signal(Vec<PythonType>),
    ContainerStack,
    SceneNode,
    SettingValue,
}

pub fn map_to_rust(py_type: &PythonType) -> RustType {
    match py_type {
        PythonType::Int => RustType::I64,
        PythonType::Float => RustType::F64,
        PythonType::Bool => RustType::Bool,
        PythonType::Str => RustType::String,
        PythonType::Bytes => RustType::VecU8,
        PythonType::None => RustType::Unit,

        PythonType::List(inner) => RustType::Vec(Box::new(map_to_rust(inner))),
        PythonType::Dict(k, v) => RustType::HashMap(
            Box::new(map_to_rust(k)),
            Box::new(map_to_rust(v))
        ),
        PythonType::Set(inner) => RustType::HashSet(Box::new(map_to_rust(inner))),
        PythonType::Optional(inner) => RustType::Option(Box::new(map_to_rust(inner))),

        PythonType::Signal(args) => RustType::BroadcastSender(
            args.iter().map(map_to_rust).collect()
        ),

        // ... etc
    }
}
```

---

## Stage 3: Intermediate Representation (IR)

### IR Design

The IR is a Rust-oriented representation that captures semantics without Python-specific constructs:

```rust
pub enum IrModule {
    Library {
        name: String,
        visibility: Visibility,
        items: Vec<IrItem>,
    },
    Binary {
        name: String,
        main: IrFunction,
        modules: Vec<IrModule>,
    },
}

pub enum IrItem {
    Struct(IrStruct),
    Enum(IrEnum),
    Trait(IrTrait),
    Impl(IrImpl),
    Function(IrFunction),
    Const(IrConst),
    Static(IrStatic),
    TypeAlias(IrTypeAlias),
    Use(IrUse),
}

pub struct IrStruct {
    pub name: String,
    pub visibility: Visibility,
    pub generics: Vec<GenericParam>,
    pub fields: Vec<IrField>,
    pub derives: Vec<String>,
    pub attributes: Vec<Attribute>,
}

pub struct IrFunction {
    pub name: String,
    pub visibility: Visibility,
    pub is_async: bool,
    pub generics: Vec<GenericParam>,
    pub self_param: Option<SelfParam>,
    pub params: Vec<IrParam>,
    pub return_type: IrType,
    pub body: IrBlock,
    pub attributes: Vec<Attribute>,
}

pub struct IrBlock {
    pub statements: Vec<IrStatement>,
    pub expression: Option<Box<IrExpression>>,
}

pub enum IrStatement {
    Let {
        pattern: IrPattern,
        type_annotation: Option<IrType>,
        value: IrExpression,
        is_mut: bool,
    },
    Expression(IrExpression),
    Return(Option<IrExpression>),
    If {
        condition: IrExpression,
        then_block: IrBlock,
        else_block: Option<IrBlock>,
    },
    Match {
        scrutinee: IrExpression,
        arms: Vec<IrMatchArm>,
    },
    Loop(IrBlock),
    While {
        condition: IrExpression,
        body: IrBlock,
    },
    For {
        pattern: IrPattern,
        iterator: IrExpression,
        body: IrBlock,
    },
    // ... etc
}

pub enum IrExpression {
    Literal(IrLiteral),
    Variable(String),
    FieldAccess {
        base: Box<IrExpression>,
        field: String,
    },
    MethodCall {
        receiver: Box<IrExpression>,
        method: String,
        args: Vec<IrExpression>,
    },
    FunctionCall {
        function: Box<IrExpression>,
        args: Vec<IrExpression>,
    },
    Binary {
        op: BinaryOp,
        left: Box<IrExpression>,
        right: Box<IrExpression>,
    },
    Unary {
        op: UnaryOp,
        operand: Box<IrExpression>,
    },
    Closure {
        params: Vec<IrParam>,
        body: IrBlock,
        is_async: bool,
        is_move: bool,
    },
    Await(Box<IrExpression>),
    Try(Box<IrExpression>),
    // ... etc
}
```

---

## Stage 4: Transformation Passes

### Pass Architecture

Transformations are modular and composable:

```rust
pub trait TransformPass {
    fn name(&self) -> &'static str;
    fn transform(&self, ir: &mut IrModule, ctx: &mut TransformContext) -> Result<(), TransformError>;
}

pub struct TransformPipeline {
    passes: Vec<Box<dyn TransformPass>>,
}

impl TransformPipeline {
    pub fn cura_default() -> Self {
        Self {
            passes: vec![
                Box::new(passes::SignalToChannelPass),
                Box::new(passes::SingletonToLazyStaticPass),
                Box::new(passes::DecoratorToTraitPass),
                Box::new(passes::QtPropertyPass),
                Box::new(passes::ContainerStackPass),
                Box::new(passes::ExceptionToResultPass),
                Box::new(passes::AsyncifyPass),
                Box::new(passes::LifetimeInferencePass),
                Box::new(passes::BorrowCheckPrePass),
            ],
        }
    }

    pub fn run(&self, ir: &mut IrModule) -> Result<TransformReport, TransformError> {
        let mut ctx = TransformContext::new();
        for pass in &self.passes {
            pass.transform(ir, &mut ctx)?;
        }
        Ok(ctx.into_report())
    }
}
```

### Key Transformation Passes

#### 1. Signal → Channel Pass

Converts Qt signals to tokio broadcast channels:

```rust
pub struct SignalToChannelPass;

impl TransformPass for SignalToChannelPass {
    fn transform(&self, ir: &mut IrModule, ctx: &mut TransformContext) -> Result<(), TransformError> {
        for item in &mut ir.items {
            if let IrItem::Struct(s) = item {
                let signals: Vec<_> = s.fields.iter()
                    .filter(|f| matches!(&f.ty, IrType::Signal(_)))
                    .cloned()
                    .collect();

                for signal in signals {
                    // Replace signal field with channel sender
                    self.convert_signal_field(s, &signal)?;

                    // Add subscribe method
                    self.add_subscribe_method(s, &signal)?;

                    // Convert all .emit() calls to .send()
                    self.convert_emit_calls(ir, &s.name, &signal)?;

                    // Convert all .connect() calls to .subscribe()
                    self.convert_connect_calls(ir, &s.name, &signal)?;
                }
            }
        }
        Ok(())
    }
}
```

**Example transformation**:

```python
# Python input
class MachineManager(QObject):
    globalContainerChanged = pyqtSignal()

    def setActiveMachine(self, stack):
        self._stack = stack
        self.globalContainerChanged.emit()
```

```rust
// Rust output
pub struct MachineManager {
    global_container_changed_tx: broadcast::Sender<()>,
    stack: Option<GlobalStack>,
}

impl MachineManager {
    pub fn set_active_machine(&mut self, stack: GlobalStack) {
        self.stack = Some(stack);
        let _ = self.global_container_changed_tx.send(());
    }

    pub fn on_global_container_changed(&self) -> broadcast::Receiver<()> {
        self.global_container_changed_tx.subscribe()
    }
}
```

#### 2. Singleton → Lazy Static Pass

```rust
pub struct SingletonToLazyStaticPass;

impl TransformPass for SingletonToLazyStaticPass {
    fn transform(&self, ir: &mut IrModule, ctx: &mut TransformContext) -> Result<(), TransformError> {
        let singletons = self.detect_singleton_patterns(ir);

        for singleton in singletons {
            // Add static lazy instance
            ir.items.push(IrItem::Static(IrStatic {
                name: format!("{}_INSTANCE", singleton.name.to_uppercase()),
                ty: IrType::Lazy(Box::new(IrType::Arc(Box::new(
                    IrType::RwLock(Box::new(IrType::Named(singleton.name.clone())))
                )))),
                value: Some(self.generate_lazy_init(&singleton)),
                visibility: Visibility::Private,
            }));

            // Add instance() class method
            self.add_instance_method(&mut ir.items, &singleton)?;

            // Convert getInstance() calls to instance()
            self.convert_get_instance_calls(ir, &singleton)?;
        }
        Ok(())
    }

    fn detect_singleton_patterns(&self, ir: &IrModule) -> Vec<SingletonInfo> {
        // Look for:
        // - __instance class variable set to None
        // - getInstance() classmethod that creates if None
        // - Private __init__ or special construction pattern
    }
}
```

#### 3. Decorator Pattern Pass

Converts Python's dynamic decorator pattern to Rust traits:

```rust
pub struct DecoratorToTraitPass;

impl TransformPass for DecoratorToTraitPass {
    fn transform(&self, ir: &mut IrModule, ctx: &mut TransformContext) -> Result<(), TransformError> {
        // Identify decorator base class (SceneNodeDecorator)
        let decorator_bases = self.find_decorator_bases(ir);

        for base in decorator_bases {
            // Generate trait from decorator methods
            let trait_def = self.generate_decorator_trait(&base);
            ir.items.push(IrItem::Trait(trait_def));

            // For each decorator implementation, generate impl
            for decorator_impl in self.find_decorator_impls(ir, &base) {
                let impl_block = self.generate_trait_impl(&decorator_impl, &base);
                ir.items.push(IrItem::Impl(impl_block));
            }

            // Convert callDecoration() to trait object dispatch
            self.convert_call_decoration(ir, &base)?;
        }
        Ok(())
    }
}
```

#### 4. Exception → Result Pass

```rust
pub struct ExceptionToResultPass;

impl TransformPass for ExceptionToResultPass {
    fn transform(&self, ir: &mut IrModule, ctx: &mut TransformContext) -> Result<(), TransformError> {
        for item in &mut ir.items {
            if let IrItem::Function(f) = item {
                // Detect if function can raise exceptions
                let can_fail = self.analyze_exception_paths(&f.body);

                if can_fail {
                    // Wrap return type in Result
                    f.return_type = IrType::Result(
                        Box::new(f.return_type.clone()),
                        Box::new(IrType::Named("anyhow::Error".to_string()))
                    );

                    // Convert raise to return Err()
                    self.convert_raise_statements(&mut f.body)?;

                    // Add ? to fallible calls
                    self.add_question_marks(&mut f.body)?;
                }
            }
        }
        Ok(())
    }
}
```

#### 5. Container Stack Pass (Cura-Specific)

```rust
pub struct ContainerStackPass;

impl TransformPass for ContainerStackPass {
    fn transform(&self, ir: &mut IrModule, ctx: &mut TransformContext) -> Result<(), TransformError> {
        // Recognize CuraContainerStack pattern
        // - Ordered container hierarchy
        // - Property resolution through stack
        // - Signal propagation

        for item in &mut ir.items {
            if let IrItem::Struct(s) = item {
                if self.is_container_stack(&s) {
                    // Generate proper container fields with lifetimes
                    self.restructure_container_fields(s)?;

                    // Add proper getter methods with resolution
                    self.add_property_resolution_methods(s)?;

                    // Handle container change propagation
                    self.add_change_notification(s)?;
                }
            }
        }
        Ok(())
    }
}
```

---

## Stage 5: Code Generation

### Rust Code Emitter

Uses `syn` and `quote` for code generation:

```rust
use proc_macro2::TokenStream;
use quote::{quote, format_ident};
use syn;

pub struct RustCodeGenerator {
    config: GeneratorConfig,
}

impl RustCodeGenerator {
    pub fn generate_module(&self, ir: &IrModule) -> TokenStream {
        let items: Vec<TokenStream> = ir.items.iter()
            .map(|item| self.generate_item(item))
            .collect();

        quote! {
            #(#items)*
        }
    }

    fn generate_struct(&self, s: &IrStruct) -> TokenStream {
        let name = format_ident!("{}", s.name);
        let vis = self.generate_visibility(&s.visibility);
        let derives = self.generate_derives(&s.derives);
        let fields = s.fields.iter().map(|f| self.generate_field(f));

        quote! {
            #derives
            #vis struct #name {
                #(#fields),*
            }
        }
    }

    fn generate_function(&self, f: &IrFunction) -> TokenStream {
        let name = format_ident!("{}", to_snake_case(&f.name));
        let vis = self.generate_visibility(&f.visibility);
        let async_kw = if f.is_async { quote!(async) } else { quote!() };
        let params = f.params.iter().map(|p| self.generate_param(p));
        let return_type = self.generate_type(&f.return_type);
        let body = self.generate_block(&f.body);

        quote! {
            #vis #async_kw fn #name(#(#params),*) -> #return_type {
                #body
            }
        }
    }

    // ... etc
}
```

### Post-Processing

```rust
pub struct PostProcessor {
    config: PostProcessConfig,
}

impl PostProcessor {
    pub fn process(&self, generated: &str) -> Result<String, PostProcessError> {
        let mut output = generated.to_string();

        // Format with rustfmt
        output = self.run_rustfmt(&output)?;

        // Run clippy and apply auto-fixes
        if self.config.auto_clippy_fix {
            output = self.run_clippy_fix(&output)?;
        }

        // Add file header
        output = self.add_header(&output);

        Ok(output)
    }

    fn add_header(&self, code: &str) -> String {
        format!(
            r#"// Auto-generated by py2cura-rs from Python source
// Source version: {}
// Generated: {}
// DO NOT EDIT MANUALLY - changes will be overwritten
// Use patches/ for manual modifications

{}"#,
            self.config.source_version,
            chrono::Utc::now().format("%Y-%m-%d %H:%M:%S UTC"),
            code
        )
    }
}
```

---

## Stage 6: Patch System

### Patch Format

Patches are stored as unified diffs with metadata:

```
patches/
├── manifest.toml
├── 0001-fix-container-lifetime.patch
├── 0002-add-send-sync-bounds.patch
└── 0003-optimize-settings-cache.patch
```

```toml
# patches/manifest.toml

[[patches]]
id = "0001"
name = "fix-container-lifetime"
description = "Add explicit lifetime to container references"
applies_to = ["cura-settings/src/container_stack.rs"]
upstream_version = "5.11.0"
author = "developer@example.com"
date = "2026-01-15"

[[patches]]
id = "0002"
name = "add-send-sync-bounds"
description = "Add Send+Sync bounds for thread safety"
applies_to = ["cura-scene/src/node.rs"]
upstream_version = "5.11.0"
```

### Patch Application

```rust
pub struct PatchManager {
    patches_dir: PathBuf,
    manifest: PatchManifest,
}

impl PatchManager {
    pub fn apply_all(&self, target_dir: &Path) -> Result<PatchReport, PatchError> {
        let mut report = PatchReport::new();

        for patch in &self.manifest.patches {
            match self.apply_patch(target_dir, patch) {
                Ok(applied) => report.add_success(patch.id.clone(), applied),
                Err(e) => {
                    if patch.required {
                        return Err(e);
                    }
                    report.add_failure(patch.id.clone(), e);
                }
            }
        }

        Ok(report)
    }

    pub fn create_patch(&self, original: &Path, modified: &Path, name: &str) -> Result<PathBuf, PatchError> {
        // Generate unified diff
        let diff = self.generate_diff(original, modified)?;

        // Create patch file
        let patch_id = self.next_patch_id();
        let filename = format!("{:04}-{}.patch", patch_id, name);
        let patch_path = self.patches_dir.join(&filename);

        fs::write(&patch_path, &diff)?;

        // Update manifest
        self.add_to_manifest(patch_id, name, &filename)?;

        Ok(patch_path)
    }
}
```

---

## Stage 7: Upstream Tracking

### Version Diffing

```rust
pub struct UpstreamTracker {
    upstream_repo: PathBuf,
    local_python: PathBuf,
    local_rust: PathBuf,
}

impl UpstreamTracker {
    pub fn generate_diff_report(&self, old_version: &str, new_version: &str) -> DiffReport {
        // Checkout old version
        self.checkout_version(old_version);
        let old_ast = self.parse_all_python();

        // Checkout new version
        self.checkout_version(new_version);
        let new_ast = self.parse_all_python();

        // Generate structural diff
        DiffReport {
            added_files: self.find_added_files(&old_ast, &new_ast),
            removed_files: self.find_removed_files(&old_ast, &new_ast),
            modified_files: self.find_modified_files(&old_ast, &new_ast),
            added_classes: self.find_added_classes(&old_ast, &new_ast),
            removed_classes: self.find_removed_classes(&old_ast, &new_ast),
            modified_methods: self.find_modified_methods(&old_ast, &new_ast),
            signature_changes: self.find_signature_changes(&old_ast, &new_ast),
            new_dependencies: self.find_new_dependencies(&old_ast, &new_ast),
        }
    }

    pub fn generate_migration_guide(&self, report: &DiffReport) -> String {
        let mut guide = String::new();

        guide.push_str("# Migration Guide\n\n");

        if !report.added_classes.is_empty() {
            guide.push_str("## New Classes to Port\n\n");
            for class in &report.added_classes {
                guide.push_str(&format!("- `{}` in `{}`\n", class.name, class.file));
            }
        }

        if !report.signature_changes.is_empty() {
            guide.push_str("\n## API Signature Changes\n\n");
            for change in &report.signature_changes {
                guide.push_str(&format!(
                    "### `{}.{}`\n\nOld: `{}`\nNew: `{}`\n\n",
                    change.class, change.method, change.old_sig, change.new_sig
                ));
            }
        }

        // ... etc

        guide
    }
}
```

### Automated Update Workflow

```rust
pub struct UpdatePipeline {
    tracker: UpstreamTracker,
    transpiler: Transpiler,
    patch_manager: PatchManager,
    test_runner: TestRunner,
}

impl UpdatePipeline {
    pub fn update_to_version(&self, new_version: &str) -> Result<UpdateResult, UpdateError> {
        // 1. Generate diff report
        let current_version = self.get_current_version();
        let diff = self.tracker.generate_diff_report(&current_version, new_version);

        // 2. Transpile new Python code
        let transpiled = self.transpiler.transpile_all()?;

        // 3. Apply existing patches
        let patch_result = self.patch_manager.apply_all(&transpiled)?;

        // 4. Run tests to detect regressions
        let test_result = self.test_runner.run_all()?;

        // 5. Generate report
        Ok(UpdateResult {
            diff_report: diff,
            patch_report: patch_result,
            test_report: test_result,
            manual_review_needed: self.identify_manual_review_items(&diff, &test_result),
        })
    }
}
```

---

## CLI Interface

```
py2cura-rs 1.0.0
Ultimaker Cura Python→Rust Transpiler

USAGE:
    py2cura-rs <COMMAND>

COMMANDS:
    transpile    Transpile Python source to Rust
    update       Update from new upstream version
    patch        Manage patches
    diff         Generate diff between versions
    validate     Validate transpiled code
    help         Print help information

---

py2cura-rs transpile [OPTIONS]

OPTIONS:
    -i, --input <PATH>      Python source directory [default: ./cura]
    -o, --output <PATH>     Rust output directory [default: ./cura-rs]
    -c, --config <FILE>     Configuration file [default: ./transpile.toml]
    --passes <PASSES>       Comma-separated list of transform passes
    --skip-passes <PASSES>  Passes to skip
    --dry-run               Show what would be generated without writing
    --verbose               Verbose output

---

py2cura-rs update [OPTIONS]

OPTIONS:
    --upstream <REF>        Git ref for upstream version (tag, branch, commit)
    --output <PATH>         Output directory [default: ./cura-rs]
    --auto-patch            Automatically apply patches
    --test                  Run tests after update
    --report <FILE>         Write migration report to file

---

py2cura-rs patch <SUBCOMMAND>

SUBCOMMANDS:
    list                    List all patches
    apply                   Apply patches to generated code
    create                  Create new patch from diff
    remove                  Remove a patch
    reorder                 Change patch application order

---

py2cura-rs diff [OPTIONS]

OPTIONS:
    --old <VERSION>         Old version to compare
    --new <VERSION>         New version to compare
    --output <FILE>         Output file for report [default: stdout]
    --format <FORMAT>       Output format: text, json, markdown [default: markdown]
```

---

## Configuration Reference

```toml
# transpile.toml - Full configuration reference

[general]
# Source Python version being transpiled
source_version = "5.11.0"
# Target Rust edition
target_edition = "2021"
# Generate async code by default
async_by_default = true
# Enable verbose logging
verbose = false

[paths]
# Python source directories to transpile
python_sources = ["cura/", "plugins/CuraEngineBackend/", "plugins/3MFReader/"]
# Output Rust workspace root
rust_output = "./cura-rs"
# Patches directory
patches = "./patches"

[type_mappings]
# Custom type mappings (Python fully qualified name → Rust type)
"PyQt6.QtCore.pyqtSignal" = "tokio::sync::broadcast::Sender"
"UM.Signal.Signal" = "tokio::sync::broadcast::Sender"
"UM.Scene.SceneNode.SceneNode" = "cura_scene::SceneNode"
"UM.Settings.ContainerStack.ContainerStack" = "cura_settings::ContainerStack"

[module_mappings]
# Map Python modules to Rust crates
"cura.Settings" = "cura_settings"
"cura.Scene" = "cura_scene"
"cura.Machines" = "cura_machines"
"plugins.CuraEngineBackend" = "cura_engine_bridge"
"plugins.3MFReader" = "cura_formats"

[skip]
# Modules to skip entirely
modules = [
    "cura.UI",
    "cura.OAuth2",
    "cura.UltimakerCloud",
]
# Files to skip
files = [
    "**/test_*.py",
    "**/*_test.py",
]
# Classes to skip
classes = [
    "*ViewModel",
    "*Dialog",
]

[passes]
# Enable/disable transform passes
signal_to_channel = true
singleton_to_lazy = true
decorator_to_trait = true
exception_to_result = true
asyncify = true
lifetime_inference = true

[passes.signal_to_channel]
# Channel buffer size for signals
buffer_size = 16
# Whether to make channels bounded
bounded = true

[passes.singleton_to_lazy]
# Use parking_lot instead of std mutex
use_parking_lot = true

[passes.exception_to_result]
# Error type to use
error_type = "anyhow::Error"

[codegen]
# Generate documentation comments from Python docstrings
generate_docs = true
# Add #[derive(Debug)] to all structs
derive_debug = true
# Add #[derive(Clone)] where possible
derive_clone = true
# Format generated code with rustfmt
rustfmt = true
# Clippy lint level
clippy_level = "warn"

[patches]
# Directory containing patch files
directory = "./patches"
# Auto-apply patches after transpilation
auto_apply = true
# Fail if patch doesn't apply cleanly
strict = false
```

---

## Example Transformations

### Complete Class Transformation

**Python Input:**
```python
from PyQt6.QtCore import QObject, pyqtSignal, pyqtProperty
from typing import Optional, List, Dict

class MachineManager(QObject):
    """Manages the active machine configuration."""

    __instance = None
    globalContainerChanged = pyqtSignal()
    activeMaterialChanged = pyqtSignal(str)

    def __init__(self, parent=None):
        super().__init__(parent)
        self._global_container_stack: Optional[GlobalStack] = None
        self._material_cache: Dict[str, Material] = {}

    @classmethod
    def getInstance(cls) -> "MachineManager":
        if cls.__instance is None:
            cls.__instance = MachineManager()
        return cls.__instance

    def setActiveMachine(self, stack: GlobalStack) -> None:
        """Set the active machine and notify listeners."""
        if self._global_container_stack == stack:
            return
        self._global_container_stack = stack
        self.globalContainerChanged.emit()

    @pyqtProperty(str, notify=activeMaterialChanged)
    def activeMaterialId(self) -> str:
        if self._global_container_stack is None:
            return ""
        return self._global_container_stack.material.getId()

    def getMaterials(self) -> List[Material]:
        """Get all available materials for the current machine."""
        if self._global_container_stack is None:
            return []
        return list(self._material_cache.values())
```

**Rust Output:**
```rust
// Auto-generated by py2cura-rs from Python source
// Source version: 5.11.0
// Generated: 2026-02-05 10:30:00 UTC

use std::collections::HashMap;
use std::sync::Arc;
use once_cell::sync::Lazy;
use parking_lot::RwLock;
use tokio::sync::broadcast;

use crate::global_stack::GlobalStack;
use crate::material::Material;

/// Global singleton instance
static MACHINE_MANAGER_INSTANCE: Lazy<Arc<RwLock<MachineManager>>> =
    Lazy::new(|| Arc::new(RwLock::new(MachineManager::new())));

/// Manages the active machine configuration.
#[derive(Debug)]
pub struct MachineManager {
    global_container_stack: Option<Arc<GlobalStack>>,
    material_cache: HashMap<String, Material>,

    // Signal channels
    global_container_changed_tx: broadcast::Sender<()>,
    active_material_changed_tx: broadcast::Sender<String>,
}

impl MachineManager {
    /// Create a new MachineManager
    pub fn new() -> Self {
        let (global_container_changed_tx, _) = broadcast::channel(16);
        let (active_material_changed_tx, _) = broadcast::channel(16);

        Self {
            global_container_stack: None,
            material_cache: HashMap::new(),
            global_container_changed_tx,
            active_material_changed_tx,
        }
    }

    /// Get the singleton instance
    pub fn instance() -> Arc<RwLock<MachineManager>> {
        Arc::clone(&MACHINE_MANAGER_INSTANCE)
    }

    /// Subscribe to global container changes
    pub fn on_global_container_changed(&self) -> broadcast::Receiver<()> {
        self.global_container_changed_tx.subscribe()
    }

    /// Subscribe to active material changes
    pub fn on_active_material_changed(&self) -> broadcast::Receiver<String> {
        self.active_material_changed_tx.subscribe()
    }

    /// Set the active machine and notify listeners.
    pub fn set_active_machine(&mut self, stack: GlobalStack) {
        let stack = Arc::new(stack);

        if let Some(current) = &self.global_container_stack {
            if Arc::ptr_eq(current, &stack) {
                return;
            }
        }

        self.global_container_stack = Some(stack);
        let _ = self.global_container_changed_tx.send(());
    }

    /// Get the active material ID
    pub fn active_material_id(&self) -> String {
        self.global_container_stack
            .as_ref()
            .map(|stack| stack.material().get_id())
            .unwrap_or_default()
    }

    /// Get all available materials for the current machine.
    pub fn get_materials(&self) -> Vec<Material> {
        if self.global_container_stack.is_none() {
            return Vec::new();
        }
        self.material_cache.values().cloned().collect()
    }
}

impl Default for MachineManager {
    fn default() -> Self {
        Self::new()
    }
}
```

---

## Testing Strategy

### Unit Tests for Transforms

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_signal_to_channel_transform() {
        let python_code = r#"
class Foo(QObject):
    mySignal = pyqtSignal()

    def trigger(self):
        self.mySignal.emit()
"#;

        let rust_code = transpile(python_code);

        assert!(rust_code.contains("broadcast::Sender<()>"));
        assert!(rust_code.contains(".send(())"));
        assert!(rust_code.contains("fn on_my_signal(&self) -> broadcast::Receiver<()>"));
    }

    #[test]
    fn test_singleton_transform() {
        let python_code = r#"
class Singleton:
    __instance = None

    @classmethod
    def getInstance(cls):
        if cls.__instance is None:
            cls.__instance = Singleton()
        return cls.__instance
"#;

        let rust_code = transpile(python_code);

        assert!(rust_code.contains("static SINGLETON_INSTANCE: Lazy<"));
        assert!(rust_code.contains("pub fn instance()"));
    }
}
```

### Integration Tests

```rust
#[test]
fn test_full_settings_module_transpilation() {
    let python_dir = Path::new("test_fixtures/cura/Settings");
    let output_dir = tempdir().unwrap();

    let result = transpile_module(python_dir, output_dir.path());

    assert!(result.is_ok());

    // Verify generated Rust compiles
    let compile_result = Command::new("cargo")
        .arg("check")
        .current_dir(output_dir.path())
        .status()
        .unwrap();

    assert!(compile_result.success());
}
```

---

## Future Enhancements

1. **Machine Learning-Assisted Type Inference**: Train a model on typed Python codebases to improve inference accuracy

2. **Incremental Transpilation**: Only re-transpile changed files for faster iteration

3. **IDE Integration**: VSCode extension showing Python↔Rust correspondence

4. **Bidirectional Sync**: Optionally propagate manual Rust changes back to Python (for hybrid operation period)

5. **Plugin Support**: Allow custom transform passes for specific patterns

---

*Specification Version: 1.0*
*Last Updated: 2026-02-05*
