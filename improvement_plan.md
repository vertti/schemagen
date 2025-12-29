# Schemagen - Improvement Plan

This document outlines potential improvements sorted from simplest to most complex. All changes are focused on code quality and maintainability, not functionality changes.

---

## Legend

| Complexity | Description | Estimated Effort |
|------------|-------------|-----------------|
| 🟢 Simple | Trivial changes, minimal risk | < 30 mins |
| 🟡 Moderate | Requires careful refactoring | 1-3 hours |
| 🔴 Complex | Significant restructuring | 3+ hours |

---

## 🟢 Simple Improvements

### 1. Remove Duplicated `normalizeOutputStrategy` Function

**Issue:** The `normalizeOutputStrategy` function is duplicated in two places:
- `pkg/config/flags.go:79-94`
- `pkg/compare/compare.go:25-39`

**Solution:** Move to a shared location and reuse.

**Files to change:**
- `pkg/output/planner.go` - Add function here (near other strategy constants)
- `pkg/config/flags.go` - Import and use from output
- `pkg/compare/compare.go` - Import and use from output

```go
// In pkg/output/planner.go, add:
func NormalizeStrategy(strategy string) OutputStrategy {
    switch strategy {
    case "bundle":
        return StrategyBundle
    case "multifile", "multi-file":
        return StrategyMultiFile
    case "bundledeps", "bundle-deps":
        return StrategyBundleDeps
    case "bundle-per-dir":
        return StrategyBundlePerDir
    default:
        return StrategyBundle
    }
}
```

---

### 2. Extract `getLanguageExtension` to `pkg/constants`

**Issue:** Language extension mapping is duplicated in:
- `pkg/output/planner.go:313-324`
- `pkg/output/paths.go:25` (uses same function)
- `pkg/generation/pipeline.go:77-88` (similar switch)

**Solution:** Add extension constants and a helper function to `pkg/constants/languages.go`.

```go
// In pkg/constants/languages.go, add:
const (
    ExtensionTypeScript = ".ts"
    ExtensionPython     = ".py"
    ExtensionGo         = ".go"
)

func GetExtension(lang Language) string {
    switch lang {
    case LanguageTypeScript:
        return ExtensionTypeScript
    case LanguagePython:
        return ExtensionPython
    case LanguageGo:
        return ExtensionGo
    default:
        return ".txt"
    }
}
```

---

### 3. Consolidate Enum Type Analysis Logic

**Issue:** Similar enum type categorization logic exists in:
- `pkg/lang/ts/generator.go:260-273` (hasString, hasNumber, hasOther)
- `pkg/lang/py/generator.go:100-123` (same pattern)
- `pkg/lang/py/generator.go:450-466` (same pattern again)
- `pkg/lang/golang/generator.go:236-258` (analyzeEnumValueTypes)

**Solution:** Create a shared helper in `pkg/typegraph/types.go`.

```go
// In pkg/typegraph/types.go, add:
type EnumTypeInfo struct {
    HasString  bool
    HasNumber  bool
    HasOther   bool
    IsMixed    bool
    AllStrings bool
    AllNumbers bool
}

func AnalyzeEnumValues(values []EnumValue) EnumTypeInfo {
    info := EnumTypeInfo{}
    for _, v := range values {
        switch v.Value.(type) {
        case string:
            info.HasString = true
        case float64, int, int64:
            info.HasNumber = true
        default:
            info.HasOther = true
        }
    }
    info.IsMixed = (info.HasString && info.HasNumber) ||
                   (info.HasString && info.HasOther) ||
                   (info.HasNumber && info.HasOther) ||
                   info.HasOther
    info.AllStrings = info.HasString && !info.HasNumber && !info.HasOther
    info.AllNumbers = info.HasNumber && !info.HasString && !info.HasOther
    return info
}
```

---

### 4. Add Missing `isDigit` and `isLetter` to `pkg/naming`

**Issue:** Helper functions `isDigit` and `isLetter` are defined locally in:
- `pkg/lang/ts/generator_types.go:186-193`

But similar logic exists in `pkg/naming/naming.go` using `unicode.IsLetter` and `unicode.IsDigit`.

**Solution:** Ensure consistency by either:
1. Moving helpers to `pkg/naming`
2. Using `unicode` functions directly in `pkg/lang/ts`

The simpler fix is to use `unicode` package in TypeScript generator (aligns with Go generator).

---

### 5. Remove Unused `ImportSpec` in `typegraph/types.go`

**Issue:** `typegraph.ImportSpec` (types.go:104-109) duplicates `output.ImportSpec` (planner.go:36-41).

The generators use `typegraph.ImportSpec` but `output.ImportSpec` is also used. Check if one can be removed.

**Investigation needed:** Trace usage to determine if consolidation is possible.

---

## 🟡 Moderate Improvements

### 6. Extract Common Field Building Logic

**Issue:** Building fields from schema properties follows the same pattern in multiple places:
- `pkg/typegraph/builder_struct.go:69-85` (main schema properties)
- `pkg/typegraph/builder_struct.go:47-64` (allOf branch properties)
- `pkg/typegraph/builder_struct.go:237-253` (inline object extraction)
- `pkg/typegraph/builder_struct.go:279-293` (buildFieldsFromProperties)

**Solution:** Create a single private helper method:

```go
func (b *Builder) buildField(propName string, propSchema *jsonschema.Schema, requiredMap map[string]bool) *Field {
    field := &Field{
        Name:        naming.ToPascalCase(propName),
        JSONName:    propName,
        Description: getDescription(propSchema),
        Required:    requiredMap[propName],
        OmitEmpty:   !requiredMap[propName],
        Type:        b.buildTypeRef(propSchema, propName),
    }
    b.extractConstraints(field, propSchema)
    return field
}
```

---

### 7. Consolidate OneOf/AnyOf Handling in `buildTypeRef`

**Issue:** The handling of `oneOf` and `anyOf` in `pkg/typegraph/builder.go:366-434` is nearly identical code (68 lines copy-pasted).

**Current code pattern (repeated twice):**
```go
if len(schema.OneOf) > 0 {  // OR schema.AnyOf
    ref.Kind = KindUnion
    ref.UnionMembers = make([]*TypeRef, 0, len(schema.OneOf))
    for i, memberSchema := range schema.OneOf {
        if b.shouldExtractInlineObject(memberSchema) {
            // ... 20 lines of extraction logic
        } else {
            memberRef := b.buildTypeRef(memberSchema, "")
            ref.UnionMembers = append(ref.UnionMembers, memberRef)
        }
    }
    return ref
}
```

**Solution:** Extract to a helper method:

```go
func (b *Builder) buildUnionRef(members []*jsonschema.Schema, fieldName string) *TypeRef {
    ref := &TypeRef{
        Kind:         KindUnion,
        UnionMembers: make([]*TypeRef, 0, len(members)),
    }
    for i, memberSchema := range members {
        if b.shouldExtractInlineObject(memberSchema) {
            // extraction logic
        } else {
            memberRef := b.buildTypeRef(memberSchema, "")
            ref.UnionMembers = append(ref.UnionMembers, memberRef)
        }
    }
    return ref
}

// Then in buildTypeRef:
if len(schema.OneOf) > 0 {
    return b.buildUnionRef(schema.OneOf, fieldName)
}
if len(schema.AnyOf) > 0 {
    return b.buildUnionRef(schema.AnyOf, fieldName)
}
```

---

### 8. Unify Type Reference Collection Functions

**Issue:** Similar recursive type reference collection exists in:
- `pkg/output/planner.go:197-242` - `collectReferencedTypes`, `collectFieldReferences`
- `pkg/output/planner.go:245-291` - `collectOrphanedTypes`, `collectOrphanedFieldReferences`
- `pkg/output/imports.go:80-126` - `collectTypeReferences`, `collectTypeRefReferences`
- `pkg/lang/py/generator_types.go:193-219` - `collectTypeDependencies`

**Solution:** Create a generic walker in `pkg/typegraph`:

```go
// In pkg/typegraph/types.go or new file:
type TypeRefVisitor func(typeName string)

func (ref *TypeRef) Walk(visitor TypeRefVisitor) {
    if ref == nil {
        return
    }
    if ref.TypeName != "" {
        visitor(ref.TypeName)
    }
    for _, member := range ref.UnionMembers {
        member.Walk(visitor)
    }
    if ref.ItemType != nil {
        ref.ItemType.Walk(visitor)
    }
    if ref.ValueType != nil {
        ref.ValueType.Walk(visitor)
    }
    for _, field := range ref.ObjectFields {
        field.Type.Walk(visitor)
    }
}

func (t *Type) WalkReferences(visitor TypeRefVisitor) {
    for _, field := range t.Fields {
        field.Type.Walk(visitor)
    }
    for _, base := range t.Extends {
        visitor(base)
    }
    // etc.
}
```

---

### 9. Consolidate JSDoc/Comment Generation

**Issue:** JSDoc generation pattern is repeated in TypeScript generator:
- `pkg/lang/ts/generator.go:117-121` (type description)
- `pkg/lang/ts/generator.go:169-189` (field with format)
- `pkg/lang/ts/generator.go:253-257` (enum description)
- `pkg/lang/ts/generator.go:319-323` (primitive alias)
- Similar in `pkg/lang/ts/generator_types.go:121-140`

**Solution:** Create helper functions:

```go
func formatJSDoc(description string, format string) string {
    if description == "" && format == "" {
        return ""
    }
    var lines []string
    lines = append(lines, "/**")
    if description != "" {
        lines = append(lines, fmt.Sprintf(" * %s", description))
    }
    if format != "" {
        lines = append(lines, fmt.Sprintf(" * @format %s", format))
    }
    lines = append(lines, " */")
    return strings.Join(lines, "\n")
}
```

---

### 10. Extract Barrel File Content Generation

**Issue:** `pkg/output/barrel.go` has `GenerateBarrelFiles` and `GenerateNestedBarrels` with overlapping logic for:
- Determining barrel file paths
- Grouping files by directory
- Building export lists

**Solution:** Refactor to share a common implementation:

```go
type BarrelConfig struct {
    Files    []OutputFile
    Language string
    Nested   bool  // Include subdirectory re-exports
}

func GenerateBarrels(cfg BarrelConfig) []BarrelFile {
    // Unified implementation
}
```

---

## 🔴 Complex Improvements

### 11. Introduce Type Renderer Interface

**Issue:** Each generator has similar but slightly different type-to-string conversion methods:
- `pkg/lang/ts/generator_types.go:19-97` - `typeRefToTS`
- `pkg/lang/py/generator_types.go:68-155` - `typeRefToPython`
- `pkg/lang/golang/generator_types.go:20-51` - `typeRefToGoType`

These all follow the same switch-on-Kind pattern but with language-specific output.

**Solution:** Consider a visitor or strategy pattern:

```go
type TypeRenderer interface {
    RenderRef(name string) string
    RenderEnum(values []interface{}) string
    RenderUnion(members []string) string
    RenderPrimitive(goType string) string
    RenderArray(itemType string) string
    RenderMap(valueType string) string
    RenderAny() string
    WrapNullable(typStr string) string
}

func RenderTypeRef(ref *TypeRef, r TypeRenderer) string {
    // Common logic calling renderer methods
}
```

**Trade-off:** This adds abstraction complexity. May not be worth it unless adding more languages.

---

### 12. Split Large `typegraph.Type` Struct

**Issue:** `pkg/typegraph/types.go` has `Type` struct with 15+ fields for all type kinds, where only a subset applies to each kind:
- Struct: Fields, Extends, AdditionalProps
- Enum: EnumType, EnumValues, HasComplexValues
- Primitive: GoType
- Array: ItemType
- Union: UnionMembers
- etc.

**Solution:** Consider a discriminated union approach:

```go
type Type struct {
    ID          string
    Name        string
    Kind        TypeKind
    Description string
    Data        TypeData  // interface{}
}

type StructData struct {
    Fields          []*Field
    Extends         []string
    AdditionalProps *AdditionalPropsConfig
}

type EnumData struct {
    EnumType         string
    Values           []EnumValue
    HasComplexValues bool
}

// etc.
```

**Trade-off:** Requires type assertions at usage sites. Current flat struct is simpler despite wasted memory.

**Recommendation:** Keep current design unless memory is a concern with very large schemas.

---

### 13. Unify Import Handling Across Generators

**Issue:** Each generator adapter handles import conversion differently:
- `pkg/generation/typescript.go:38-47` - Direct conversion
- `pkg/generation/python.go:43-62` - Path transformation (. instead of /)
- `pkg/generation/golang.go:45-61` - Module path prefixing

**Solution:** Consider moving language-specific import logic to language packages:

```go
// In pkg/lang/ts/imports.go
func ConvertImport(imp output.ImportSpec) typegraph.ImportSpec

// In pkg/lang/py/imports.go
func ConvertImport(imp output.ImportSpec) typegraph.ImportSpec

// In pkg/lang/golang/imports.go
func ConvertImport(imp output.ImportSpec, modulePath string) typegraph.ImportSpec
```

This makes import handling co-located with the generator it serves.

---

### 14. Consider Configuration Object Pattern

**Issue:** Generator configs are passed through multiple layers with similar but slightly different structures:
- `config.GenerationFlags` (CLI layer)
- `generation.Config` (pipeline layer)
- `ts.Config`, `py.Config`, `golang.Config` (generator layer)

There's conversion at each boundary.

**Solution:** Consider a unified config builder pattern:

```go
type GeneratorConfigBuilder struct {
    base *BaseConfig
}

func (b *GeneratorConfigBuilder) ForTypeScript() *ts.Config
func (b *GeneratorConfigBuilder) ForPython() *py.Config
func (b *GeneratorConfigBuilder) ForGo() *golang.Config
```

**Trade-off:** Current explicit conversion is actually clearer and allows validation at boundaries. This might be over-engineering.

---

### 15. Introduce Schema Visitor Pattern

**Issue:** `pkg/typegraph/builder.go` has many recursive methods that traverse schemas:
- `processSchema`
- `extractDefinition`
- `buildTypeRef`
- `buildStruct`

**Solution:** A visitor pattern could make this more extensible:

```go
type SchemaVisitor interface {
    VisitObject(schema *jsonschema.Schema) error
    VisitEnum(schema *jsonschema.Schema) error
    VisitUnion(schema *jsonschema.Schema) error
    VisitRef(ref string) error
    VisitProperty(name string, schema *jsonschema.Schema) error
}

func WalkSchema(schema *jsonschema.Schema, visitor SchemaVisitor) error {
    // Traverse and call appropriate visitor methods
}
```

**Trade-off:** Adds significant complexity. Only worthwhile if you need to add new schema processors frequently.

---

## Summary by Priority

### High Priority (Quick Wins)
1. ✅ Remove duplicated `normalizeOutputStrategy` - 10 mins
2. ✅ Extract `getLanguageExtension` to constants - 15 mins
3. ✅ Consolidate enum type analysis logic - 30 mins

### Medium Priority (Good ROI)
4. Extract common field building logic - 1 hour
5. Consolidate oneOf/anyOf handling - 45 mins
6. Consolidate JSDoc generation - 30 mins

### Low Priority (Nice to Have)
7. Unify type reference collection - 2 hours
8. Extract barrel content generation - 1.5 hours
9. Introduce TypeRenderer interface - 3+ hours

### Consider Later (Trade-offs)
10. Split Type struct - Significant change, unclear benefit
11. Unified import handling - Current approach is fine
12. Config builder pattern - Current approach is explicit and clear
13. Schema visitor pattern - Only if adding new processors

---

## Implementation Notes

When implementing these changes:

1. **Run tests after each change** - The project has good test coverage
2. **Update golden test files** if output format changes
3. **Keep commits small and focused** - One improvement per commit
4. **Maintain backward compatibility** - Don't change CLI flags or output format

The project already follows good practices:
- Clean package boundaries
- Consistent naming conventions
- Good error handling with custom error types
- Comprehensive test coverage

These improvements focus on reducing duplication and improving maintainability, not adding features.
