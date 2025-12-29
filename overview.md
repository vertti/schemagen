# Schemagen - Architecture Overview

## Project Summary

**Schemagen** is a CLI tool that generates TypeScript, Python (Pydantic v2), and Go code from JSON Schema specifications. It is a single-binary, zero-dependency code generator that supports multiple output strategies and language-specific features.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CLI Layer                                   │
│  cmd/root.go → cmd/generate.go, cmd/validate.go, cmd/verify.go, etc.   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Schema Loading                                 │
│                    pkg/schema/loader.go, order.go                        │
│         - Load JSON/YAML files                                          │
│         - Compile with jsonschema library                                │
│         - Extract property order for deterministic output               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Type Graph Building                               │
│               pkg/typegraph/builder.go, types.go, etc.                  │
│         - Transform JSON Schema → Language-agnostic Type Graph          │
│         - Handle allOf, oneOf, anyOf, $ref, enums                       │
│         - Extract inline types when needed                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Output Planning                                 │
│              pkg/output/planner.go, imports.go, paths.go                │
│         - Determine file distribution strategy                          │
│         - Compute cross-file imports                                    │
│         - Generate barrel/index files                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Code Generation                                  │
│  pkg/generation/pipeline.go → pkg/lang/{ts,py,golang}/generator.go     │
│         - Language-specific code emission                               │
│         - Type mapping, constraints, imports                            │
│         - File writing                                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Package Structure

### `/cmd` - CLI Commands

| File | Purpose |
|------|---------|
| `root.go:16-45` | Root command setup, global flags (--verbose, --json) |
| `generate.go:13-85` | Main generation command, orchestrates multi-language output |
| `validate.go` | Schema validation command (exit 0=valid, 1=invalid) |
| `verify.go` | CI verification command (exit 2=drift detected) |
| `diff.go` | Show differences between generated and existing code |
| `flags.go:8-65` | Shared flag definitions for generation commands |
| `version.go` | Version information command |

### `/pkg/schema` - Schema Loading

| File | Purpose |
|------|---------|
| `loader.go:16-245` | Load and compile JSON/YAML schema files |
| `order.go` | Extract property order from raw JSON for deterministic output |

**Key Responsibilities:**
- Support both `.json` and `.yaml/.yml` files
- YAML→JSON conversion with proper type handling
- Inject `$id` if not present for $ref resolution
- Register schemas in compiler for cross-file references
- Extract property key order from source files

### `/pkg/typegraph` - Type Graph (IR)

| File | Purpose |
|------|---------|
| `types.go:1-129` | Core type definitions (Type, TypeRef, Field, Graph) |
| `builder.go:42-735` | Main builder, $ref resolution, property ordering |
| `builder_struct.go:11-296` | Object/struct building, allOf composition |
| `builder_enum.go:10-67` | Enum type handling |
| `builder_union.go:7-24` | Union types (oneOf/anyOf) |
| `primitives.go:1-47` | Centralized Go→Language type mapping |

**Type Kinds (9 total):**
```go
KindStruct    // Objects with properties
KindEnum      // Enumerated values
KindPrimitive // string, int, bool, etc.
KindArray     // Array types
KindMap       // Map/dictionary types
KindAlias     // Type aliases
KindRef       // Reference to another type
KindUnion     // oneOf/anyOf unions
KindInterface // Empty objects or any type
```

**Key Data Structures:**
```go
type Type struct {
    ID, Name, Kind, Description
    Fields []*Field       // For structs
    Extends []string      // allOf base types
    EnumType, EnumValues  // For enums
    UnionMembers          // For unions
    // ... more for arrays, maps, aliases
}

type Field struct {
    Name, JSONName, Type, Description
    Required, OmitEmpty
    // Validation constraints:
    MinLength, MaxLength, Pattern
    Minimum, Maximum, ExclusiveMinimum, ExclusiveMaximum
    MinItems, MaxItems
}

type TypeRef struct {
    Kind, TypeName, GoType
    Nullable, Format
    ItemType, ValueType   // For arrays/maps
    UnionMembers          // For inline unions
    EnumValues            // For inline enums
    ObjectFields          // For inline objects
}
```

### `/pkg/output` - Output Planning

| File | Purpose |
|------|---------|
| `planner.go:14-325` | Output strategy planning (bundle, multifile, etc.) |
| `imports.go:1-139` | Cross-file import computation |
| `paths.go:1-130` | Path mapping and relative import calculation |
| `barrel.go:1-272` | Barrel/index file generation (TS: index.ts, Py: __init__.py) |

**Output Strategies:**
| Strategy | Description |
|----------|-------------|
| `bundle` | All types in single file (`types.ts/py/go`) |
| `multi-file` | One file per input schema |
| `bundle-deps` | Root schema + all dependencies in one file |
| `bundle-per-dir` | One file per directory |

### `/pkg/generation` - Generation Pipeline

| File | Purpose |
|------|---------|
| `config.go:1-60` | Configuration types for all languages |
| `pipeline.go:13-214` | Main generation orchestration |
| `factory.go:10-35` | Generator factory (language selection) |
| `typescript.go:1-48` | TypeScript adapter |
| `python.go:1-63` | Python adapter with import path conversion |
| `golang.go:1-97` | Go adapter with module path handling |
| `writer.go` | File writer abstraction |

### `/pkg/lang/{ts,py,golang}` - Language Generators

**TypeScript (`pkg/lang/ts/`):**
| File | Purpose |
|------|---------|
| `generator.go:42-377` | Interfaces, type aliases, JSDoc comments |
| `generator_types.go:1-194` | TypeRef→TS type string conversion |

**Python (`pkg/lang/py/`):**
| File | Purpose |
|------|---------|
| `generator.go:47-580` | Pydantic BaseModel classes, Enum classes |
| `generator_types.go:1-303` | TypeRef→Python type annotation, topological sort |
| `generator_fields.go:1-82` | Field() parameter building for Pydantic |

**Go (`pkg/lang/golang/`):**
| File | Purpose |
|------|---------|
| `generator.go:47-353` | Struct types, const/var enums |
| `generator_types.go:1-115` | TypeRef→Go type, import scanning |
| `generator_fields.go:1-90` | JSON tags, validator tags |

### Supporting Packages

| Package | Purpose |
|---------|---------|
| `pkg/naming` | Case conversion (ToPascalCase, ToSnakeCase, ToConstantCase, ToGoFieldName) |
| `pkg/common` | Shared utilities (file headers) |
| `pkg/constants` | Language constants and normalization |
| `pkg/config` | CLI flags → generation config bridge |
| `pkg/errors` | Custom error types (SchemaError, GenerationError, ValidationError) |
| `pkg/compare` | Directory comparison for verify/diff commands |
| `pkg/logger` | Structured logging (zerolog) |
| `pkg/loader` | Schema discovery and resolution |

---

## Data Flow

```
1. CLI Input
   │
   ├─► schema.Loader.Load(path)
   │   ├─► Read JSON/YAML files
   │   ├─► Convert YAML to JSON
   │   ├─► Extract property order
   │   ├─► Inject $id for refs
   │   └─► Compile with jsonschema.Compiler
   │
   ▼
2. typegraph.Builder.Build(schemas)
   │
   ├─► Process $defs (extract as types)
   ├─► Process root schemas
   │   ├─► buildStruct() - objects, allOf
   │   ├─► buildEnum() - enums
   │   ├─► buildUnion() - oneOf/anyOf
   │   └─► buildTypeRef() - recursive field processing
   ├─► Extract inline types (configurable)
   └─► Resolve $refs
   │
   ▼
3. output.PlanOutput(graph, schemas, strategy)
   │
   ├─► Determine file distribution
   ├─► Map types to files
   └─► Build type-to-file index
   │
   ▼
4. output.ComputeImports(files, typeToFile)
   │
   ├─► Track cross-file references
   └─► Calculate relative import paths
   │
   ▼
5. generation.generateFiles(graph, plan, cfg)
   │
   ├─► Create language-specific generator
   ├─► For each file:
   │   ├─► Convert imports to language format
   │   ├─► Generate code string
   │   └─► Write to disk
   └─► Generate barrel files (multifile only)
```

---

## Key Design Patterns

### 1. Language-Agnostic IR (Type Graph)
The `typegraph.Type` and `typegraph.TypeRef` structures form a language-agnostic intermediate representation. This allows:
- Single schema parsing logic
- Reusable type extraction and constraint handling
- Clean separation between parsing and code generation

### 2. Generator Interface
```go
type Generator interface {
    Generate(types []*typegraph.Type, imports interface{}) (string, error)
    ConvertImports(imports []output.ImportSpec) interface{}
}
```
Each language implements this interface, allowing pluggable generators.

### 3. Centralized Type Mapping
`pkg/typegraph/primitives.go` provides a single source of truth for Go→Language type mapping:
```go
var PrimitiveMappings = map[string]PrimitiveMapping{
    "string":      {TypeScript: "string", Python: "str", Go: "string"},
    "int":         {TypeScript: "number", Python: "int", Go: "int"},
    // ...
}
```

### 4. Property Order Preservation
Property order is extracted from raw JSON before parsing and carried through the entire pipeline to ensure deterministic output matching source file order.

### 5. Strategy Pattern for Output
Output strategies (bundle, multifile, etc.) are selected at runtime and handled by `output.PlanOutput()` without changing generator logic.

---

## Configuration Options

### Universal Flags
| Flag | Default | Description |
|------|---------|-------------|
| `--extract-inline` | false | Extract inline enums/objects to top-level types |
| `--disable-headers` | false | Disable generated file headers |
| `--disable-timestamp` | false | Disable timestamp in headers |
| `--output-strategy` | multifile | bundle, multi-file, bundle-deps |

### TypeScript Flags
| Flag | Default | Description |
|------|---------|-------------|
| `--ts-unknown-any` | false | Use `unknown` instead of `any` |
| `--ts-additional-properties` | false | Add index signatures |

### Python Flags
| Flag | Default | Description |
|------|---------|-------------|
| `--py-snake-case-field` | false | Convert fields to snake_case with alias |
| `--py-additional-properties` | false | Add `model_config = ConfigDict(extra='allow')` |

### Go Flags
| Flag | Default | Description |
|------|---------|-------------|
| `--go-package` | models | Package name |
| `--go-pointers` | true | Use pointers for optional fields |
| `--go-omit-empty` | true | Add omitempty to optional fields |
| `--go-module-path` | "" | Module path for absolute imports |

---

## Testing Strategy

The project uses:
1. **Unit Tests** - Per-package tests (`*_test.go`)
2. **Golden Tests** - Expected output in `testdata/expected/` compared against generated output
3. **Integration Tests** - Full command execution tests in `cmd/*_test.go`

Test patterns:
- `t.TempDir()` for isolated test directories
- `testify/assert` and `testify/require` for assertions
- Exit code verification for CLI commands

---

## Dependencies

| Dependency | Purpose |
|------------|---------|
| `github.com/kaptinlin/jsonschema` | JSON Schema compiler |
| `github.com/spf13/cobra` | CLI framework |
| `github.com/rs/zerolog` | Structured logging |
| `github.com/fatih/color` | Colored terminal output |
| `gopkg.in/yaml.v3` | YAML parsing |
| `github.com/stretchr/testify` | Testing assertions |
| `github.com/kylelemons/godebug` | Diff utilities |
