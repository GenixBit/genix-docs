# Genix Multi-File Source Maps

Genix preserves source identity across project/module loading so diagnostics can refer to the original `.gb` file even after functions from multiple modules are merged into one checked program.

> Status: pre-alpha. The source-map model is now implemented for project files and imported modules. Expression-level semantic spans and a structured semantic-diagnostic return type are still being generalized.

## Why source maps are required

A Genix project can contain multiple files:

```text
my-app/
├── genix.toml
└── src/
    ├── main.gb
    └── math.gb
```

The compiler currently namespaces imported functions and merges them for type checking:

```text
src/math.gb: fn add(...)
        ↓
canonical function identity: math.add
        ↓
merged checked program
```

Without a source map, a later type error knows the canonical function name but not which source file originally defined it.

The source map fixes that boundary.

## Source map model

The compiler now records mappings conceptually equivalent to:

```text
SourceMap
├── files
│   ├── src/main.gb → original source text
│   └── src/math.gb → original source text
├── functions
│   ├── main → src/main.gb
│   └── math.add → src/math.gb
├── modules
│   └── math → src/math.gb
└── entry
    └── src/main.gb
```

Source mapping is deliberately separate from executable AST/IR metadata.

```text
source files
    ↓
SourceMap ───────────────→ diagnostics / future editor tooling
    ↓
parser → AST → checker → Genix IR → backend
```

This prevents terminal/editor presentation concerns from leaking into interpreter and backend data structures.

## Multi-file semantic diagnostics

Given:

```gb
// src/main.gb
import math;

fn main() {
    print(math.bad());
}
```

and:

```gb
// src/math.gb
fn bad() -> int {
    let value: int = "wrong";
    return value;
}
```

`gb check` can now report the imported module as the primary source:

```text
error[E0201]: initializer for 'value' expected int, found string
 --> src/math.gb:2:22
   |
 2 |     let value: int = "wrong";
   |                      ^^^^^^^ type mismatch
 ::: src/main.gb:4:11
   |
 4 |     print(math.bad());
   |           ---- module referenced here
   |
   = help: change the expression or annotation so the types are compatible
```

The exact column/marker width can evolve as expression-level semantic spans become richer, but the file identities and primary/related relationship are now part of the diagnostics architecture.

## Secondary labels

A diagnostic can now carry multiple related source locations.

Current related-location uses include:

- imported module referenced from the entry source
- module function definition related to a call/signature error in another file

Related locations are rendered with `:::` to distinguish them from the primary `-->` location.

```text
 --> primary.gb:line:column
 ::: related.gb:line:column
```

This model is intentionally general enough for future diagnostics such as:

```text
value declared here
borrow originated here
function defined here
module imported here
previous declaration here
trait requirement introduced here
```

## Lexer and parser errors in modules

Project loading now uses the structured lexer/parser diagnostic path for imported files as well.

A syntax error in `src/math.gb` therefore reports `src/math.gb` directly instead of being wrapped as an unstructured project-loader string.

Import lines are replaced by blank lines before the normal parser runs, preserving downstream line numbers while imports remain handled by the project/module loader.

## Semantic mapping adapter

The current type checker still returns pre-alpha semantic errors as strings.

To avoid coupling source locations into the executable AST, the project layer identifies which function produced the semantic failure while retaining the complete project signature set. It then maps that canonical function identity through `SourceMap` and feeds the correct original source into the diagnostics engine.

This is a transitional adapter.

The intended future checker interface is conceptually:

```text
SemanticDiagnostic
├── code
├── message
├── primary SourceId + Span
├── related SourceId + Span[]
└── help
```

At that point no semantic error-text inference will be required.

## Relationship to future tooling

The source map is foundational infrastructure for:

- richer compiler diagnostics
- language-server diagnostics
- go to definition
- find references
- hover information
- rename
- debugger source mapping
- test assertion locations
- future macro/generic expansion diagnostics

The compiler should continue to use canonical symbol identities internally while `SourceMap` translates those identities back to developer-facing source locations.

## Current limitations

- Nested module imports are not implemented yet.
- Semantic expression spans are still inferred from source text rather than attached to typed semantic nodes.
- The checker still returns string errors internally.
- Related call locations currently select the first relevant module reference in the entry source rather than maintaining a full call graph.
- Test files use their separate test frontend and do not yet participate in the project source map.
- Generated/native C source maps are not implemented.

These limitations do not change program semantics; they affect diagnostic precision and tooling richness only.

---

**Genix — by GenixBit**
