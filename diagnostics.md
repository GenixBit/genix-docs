# Genix Compiler Diagnostics

Genix compiler diagnostics are designed to be stable, source-aware, and useful from the command line, editors, CI systems, and future language-server tooling.

> Status: pre-alpha. Error codes, multi-file source identity, checker-owned semantic classification, and primary/related diagnostic locations are now part of the developer-facing diagnostics contract, but individual wording may still improve before Genix 1.0.

## Diagnostic format

A compiler error can include:

```text
error[E0201]: initializer for 'age' expected int, found string
 --> src/main.gb:2:20
   |
 2 |     let age: int = "twenty";
   |                    ^^^^^^^^ type mismatch
  = help: change the expression or annotation so the types are compatible
```

The diagnostic model contains:

- stable error code
- concise message
- primary source filename
- line and column
- source span
- primary label
- zero or more related source locations
- related labels
- optional help text

Lexer and parser errors use exact token/source spans. The diagnostics engine also supports secondary labels across files, rendered with `:::` rather than the primary `-->` marker.

## Checker-native semantic diagnostics

The static type checker now constructs semantic failures structurally at the point where they are detected.

Conceptually:

```text
SemanticError
├── code
├── message
├── label
├── help
├── current canonical function
├── semantic location hint
└── optional related function
```

The checker exposes a structured diagnostic path:

```text
AST + SourceMap
      ↓
Static Type Checker
      ↓
checker-owned SemanticError
      ↓
SourceMap resolution
      ↓
Diagnostic
├── E020x code
├── primary source + span
├── primary label
├── related locations
└── help
```

This replaces the previous pre-alpha pipeline that returned a formatted type-error string and later inspected its wording to choose an error code. Project checking also no longer stubs unrelated functions and re-runs the checker to guess which merged module produced the error.

The legacy `check(...) -> Result<(), String>` entry point remains for internal compatibility with interpreter/test/bootstrap code, but it is derived from the same structured checker error. User-facing direct-file and project diagnostics use `check_diagnostic(...)`.

## Multi-file example

Given an imported module with an invalid initializer:

```gb
// src/math.gb
fn bad() -> int {
    let value: int = "wrong";
    return value;
}
```

and an entry file that uses the module:

```gb
// src/main.gb
import math;

fn main() {
    print(math.bad());
}
```

project checking can report:

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

The primary location identifies the failing source. Related locations provide context from other source files.

See `source-maps.md` for the complete mapping architecture.

## Error-code families

### Lexer — `E000x`

| Code | Meaning |
|---|---|
| `E0001` | Unexpected character or invalid single-character operator |
| `E0002` | Unterminated string literal |
| `E0003` | Invalid numeric literal |

Example:

```gb
fn main() {
    @
}
```

produces an `E0001` diagnostic at the invalid character.

### Parser and syntax — `E010x`

| Code | Meaning |
|---|---|
| `E0100` | General syntax error or expected expression/identifier/token |
| `E0101` | Invalid or unknown type syntax |
| `E0102` | Unsupported `Option` / `Result` type shape |
| `E0103` | Invalid `match` pattern |
| `E0104` | Invalid `Some` / `Ok` / `Err` / `None` constructor use |

Parser diagnostics retain the exact token span and source filename supplied by the frontend. Project-loaded modules use the same structured lexer/parser path, so syntax failures in imported `.gb` files report the imported file directly.

### Type checking — `E020x`

| Code | Meaning |
|---|---|
| `E0201` | Type mismatch or incompatible operator operands |
| `E0202` | Undefined variable or function |
| `E0203` | Mutation of an immutable binding |
| `E0204` | Invalid, missing, or non-guaranteed return |
| `E0205` | Invalid, duplicate, or non-exhaustive `match` handling |
| `E0206` | Invalid `?` error propagation |
| `E0207` | Function-call signature / argument mismatch |
| `E0208` | Invalid value use, wrapper context, or `void` use |
| `E0209` | Duplicate declaration, parameter, or function |
| `E0210` | Invalid or missing `fn main()` entry point |

These codes are assigned directly by the checker. There is no active fallback that classifies semantic errors by searching their rendered message text.

## Type mismatch example

```gb
fn main() {
    let age: int = "twenty";
}
```

The checker produces `E0201`; SourceMap resolution points the diagnostic at the initializer value.

## Undefined names

```gb
fn main() {
    print(username);
}
```

Undefined variables and functions are emitted as `E0202` by the checker.

## Mutability

```gb
fn main() {
    let count = 0;
    count = 1;
}
```

Reassigning an immutable `let` binding is `E0203`; the checker also attaches the help text recommending `mut` when reassignment is intentional.

## Call signatures and related definitions

Function-call mismatches are `E0207`.

```gb
fn add(value: int) -> int {
    return value;
}

fn main() {
    print(add(1, 2));
}
```

The semantic error records `add` as the related function. The diagnostic resolver can therefore attach the function definition as a secondary location rather than recovering the function name from a formatted error string.

For imported calls the same mechanism can cross files through canonical names such as `math.add`.

## Match exhaustiveness

```gb
fn main() {
    let value: Option<string> = None;

    match value {
        Some(text) => {
            print(text);
        }
    }
}
```

An `Option<T>` match must handle both `Some(...)` and `None`; a `Result<T,string>` match must handle both `Ok(...)` and `Err(...)`. Violations are checker-owned `E0205` errors.

## `?` propagation

`?` diagnostics use `E0206`. The operator currently requires a `Result<T,string>` expression and a surrounding function that also returns a compatible `Result`.

## Source maps

Project loading maintains a source map containing:

```text
original source files
canonical function → file
module → file
project entry file
```

This mapping survives the current module-namespacing and merge step used by the type checker.

The checker retains the canonical function identity on a semantic failure. The SourceMap then resolves that function to the correct original file without probing or re-running the checker.

Semantic expression spans are still a transitional layer: the checker emits structured location intent such as initializer, assignment, call, return, `if`, `while`, or `match`; the current resolver finds the corresponding span in the original source text. Exact source IDs/spans attached directly to semantic AST nodes are a future refinement.

## Secondary locations

Diagnostics can carry multiple related source locations.

Current uses include:

- module reference related to an error inside an imported module
- imported or local function definition related to a call/signature problem

Related locations use:

```text
 ::: file:line:column
```

This model is also intended for future declaration/borrow/trait/reference diagnostics.

## CLI behavior

These commands use structured diagnostics for direct `.gb` source files:

```bash
gb check src/main.gb
gb run src/main.gb
gb ir src/main.gb
```

Project commands use the same diagnostic model plus the multi-file source map:

```bash
gb check path/to/project
gb run path/to/project
gb build path/to/project
```

Errors discovered while loading/checking imported modules can therefore retain module file identity and related entry/module locations.

## CI contract

`genix-lang` contains both the main toolchain workflow and a dedicated Semantic Diagnostics workflow.

The semantic workflow intentionally compiles invalid programs and verifies:

```text
E0207 for a direct function-call signature mismatch
primary call-site filename/line
related function definition
E0201 inside an imported module
primary imported-module filename/line
secondary ::: entry filename/line
module referenced here
```

Rust unit tests also verify that the checker itself owns the semantic code and initializer span contract.

This prevents semantic diagnostics from silently regressing to error-text classification or merged-project function guessing.

## Current limitations

- Semantic error objects are structured, but exact expression-level source spans are not yet stored on executable AST/typed nodes.
- Some semantic spans are resolved from structured checker location hints against the original source text.
- Related call locations currently use source lookup rather than a full semantic reference graph.
- Test files use their separate test frontend and do not yet participate in the project SourceMap.
- Generated/native C source maps are not implemented.
- Machine-readable JSON output is not implemented yet.

## Design direction

The diagnostics model is intended to support future:

- stable machine-readable JSON diagnostics
- warnings and lints
- exact semantic expression source IDs/spans
- multiple primary/secondary labels
- notes and suggestions
- IDE/LSP diagnostics
- quick fixes
- go-to-definition/reference metadata
- macro / generated-source traces if those language features are added

The executable IR remains independent from CLI rendering concerns. Source metadata belongs to the frontend/source-map layer so interpreters and backends do not need to carry presentation-specific terminal state.

---

**Genix — by GenixBit**
