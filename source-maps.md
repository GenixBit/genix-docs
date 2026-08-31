# Genix Multi-File Source Maps

Genix preserves source identity across project/module loading so diagnostics can refer to the original `.gb` file even after functions from multiple modules are merged into one checked program.

> Status: pre-alpha. Multi-file `SourceMap` and checker-native structured semantic diagnostics are implemented. Exact expression-level semantic `SourceId`/`Span` attachment is the next precision step.

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

The checker retains canonical function identity on semantic failures. `SourceMap` translates that compiler identity back to the original developer-facing file.

## Source map model

The compiler records mappings conceptually equivalent to:

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

Source mapping is deliberately separate from executable IR metadata.

```text
source files
    ↓
SourceMap ───────────────────────────→ diagnostics / future editor tooling
    ↓                                      ↑
parser → AST → checker ── SemanticError ───┘
              ↓
         Genix IR → backend
```

This keeps terminal/editor presentation concerns out of interpreter and backend data structures.

## Checker-native semantic resolution

The type checker no longer relies on a post-check string-classification adapter for user-facing semantic diagnostics.

When the checker detects a semantic failure it records structured context conceptually equivalent to:

```text
SemanticError
├── code
├── message
├── label
├── help
├── canonical current function
├── semantic location hint
└── optional related function
```

The diagnostic path is:

```text
checker detects failure
        ↓
checker-owned SemanticError
        ↓
SourceMap resolution
        ├── canonical function → original file
        ├── module → original file
        └── entry source
        ↓
Diagnostic
├── primary file/span
├── primary label
├── related file/span[]
└── help
```

The previous project strategy that stubbed unrelated function bodies and re-ran the type checker to infer the failing function is removed from the semantic diagnostic path. The checker already knows the function in which it detected the failure.

## Current semantic location hints

Exact source spans are not yet stored on executable semantic nodes. Instead, the structured checker error records location intent that the SourceMap resolver applies to the original source text.

Current location intents include:

```text
function
name
initializer
assignment
function call
return
if
while
match
fallback source location
```

For example, an initializer mismatch records the variable's initializer as the semantic location. The resolver then points the primary diagnostic at that initializer in the original file.

This is intentionally narrower and safer than parsing the final error message. Error classification and function ownership are already structural; only precise expression-span lookup remains transitional.

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

`gb check` reports the imported module as the primary source:

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
  = help: change the expression or annotation so the types are compatible
```

The checker owns `E0201` and the canonical failing function `math.bad`; `SourceMap` resolves `math.bad` to `src/math.gb` and can attach the entry-file module reference as related context.

## Secondary labels

A diagnostic can carry multiple related source locations.

Current related-location uses include:

- imported module referenced from the entry source
- function definition related to a call/signature mismatch

For a call mismatch, the checker records the related callee symbol. The resolver can map that symbol back to its definition without extracting a function name from rendered text.

Related locations are rendered with `:::` to distinguish them from the primary `-->` location.

```text
 --> primary.gb:line:column
 ::: related.gb:line:column
```

The model is intentionally general enough for future diagnostics such as:

```text
value declared here
borrow originated here
function defined here
module imported here
previous declaration here
trait requirement introduced here
```

## Lexer and parser errors in modules

Project loading uses the structured lexer/parser diagnostic path for imported files as well.

A syntax error in `src/math.gb` therefore reports `src/math.gb` directly instead of being wrapped as an unstructured project-loader string.

Import lines are replaced by blank lines before the normal parser runs, preserving downstream line numbers while imports remain handled by the project/module loader.

## Relationship to direct-file diagnostics

Direct `.gb` commands create a one-file SourceMap and use the same checker-native diagnostic API as project commands.

```text
gb check source.gb
gb run source.gb
gb ir source.gb
```

Project commands use a multi-file SourceMap:

```text
gb check path/to/project
gb run path/to/project
gb build path/to/project
```

This keeps one semantic error model across direct files and projects.

## CI contract

The compiler has dedicated semantic-diagnostic CI coverage.

It verifies both:

1. a direct call-signature mismatch produces checker-owned `E0207`, a real call-site location, and a related function definition;
2. a type mismatch inside an imported module produces checker-owned `E0201`, the imported module as primary source, and the entry module reference as a secondary location.

Unit tests also verify that checker-created diagnostics retain their code, label, source identity, and expected semantic location.

## Relationship to future tooling

The source map is foundational infrastructure for:

- machine-readable compiler diagnostics
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
- Exact semantic expression `SourceId`/`Span` values are not yet stored directly on AST/typed semantic nodes.
- Semantic span resolution currently uses structured checker location hints against original source text.
- Related call/reference lookup is not yet a full semantic reference graph.
- Test files use their separate test frontend and do not yet participate in the project SourceMap.
- Generated/native C source maps are not implemented.
- Machine-readable JSON diagnostic output is not implemented yet.

These limitations affect diagnostic precision and tooling richness, not current language semantics.

## Next precision milestone

The next source-map evolution is to attach source identity directly to semantic syntax nodes so the checker can emit exact source IDs/spans without textual lookup:

```text
parsed node
├── SourceId
└── Span
      ↓
type checker
      ↓
SemanticDiagnostic
├── code
├── exact primary SourceId + Span
├── exact related SourceId + Span[]
└── help / notes
```

That model can then feed terminal output, JSON diagnostics, and future LSP clients from the same source-of-truth diagnostic object.

---

**Genix — by GenixBit**
