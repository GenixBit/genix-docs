# Genix Compiler Diagnostics

Genix compiler diagnostics are designed to be stable, source-aware, and useful from the command line, editors, CI systems, and future language-server tooling.

> Status: pre-alpha. Error codes, multi-file source identity, and primary/related diagnostic locations are now part of the developer-facing diagnostics contract, but individual wording may still improve before Genix 1.0.

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

Parser diagnostics retain the exact token span and source filename supplied by the frontend. Project-loaded modules now use the same structured lexer/parser path, so syntax failures in imported `.gb` files report the imported file directly.

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
| `E0299` | Unclassified pre-alpha type-checker error |

`E0299` is a compatibility fallback. New recurring error classes should receive a dedicated stable code instead of relying on the fallback.

## Type mismatch example

```gb
fn main() {
    let age: int = "twenty";
}
```

The CLI reports `E0201` and points at the initializer value.

## Undefined names

```gb
fn main() {
    print(username);
}
```

Undefined variables and functions are classified as `E0202`.

## Mutability

```gb
fn main() {
    let count = 0;
    count = 1;
}
```

Reassigning an immutable `let` binding is `E0203`; the help text recommends `mut` when reassignment is intentional.

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

An `Option<T>` match must handle both `Some(...)` and `None`; a `Result<T,string>` match must handle both `Ok(...)` and `Err(...)`. Violations are `E0205`.

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

Semantic errors are therefore mapped back to the correct source file after checking rather than being reported against a synthetic merged program.

The current semantic checker still returns pre-alpha string errors internally. A project-layer adapter identifies the failing function while preserving all project signatures, then maps that canonical function through the source map. The planned end state is a structured semantic checker that emits source IDs/spans directly.

## Secondary locations

Diagnostics can carry multiple related locations.

Current uses include:

- module reference related to an error inside an imported module
- imported function definition related to a call/signature problem from another file

Related locations use:

```text
 ::: file:line:column
```

This model is also intended for future declaration/borrow/trait/reference diagnostics.

## CLI behavior

These commands use the diagnostics renderer for direct `.gb` source files:

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

`genix-lang` CI intentionally compiles invalid programs and verifies that diagnostics include:

```text
error[E....]
filename:line:column
^
help:
```

The Rust compiler tests also construct a broken multi-file project and verify that the resulting project diagnostic contains:

```text
primary imported-module filename
secondary ::: entry filename
module referenced here
```

This prevents multi-file diagnostics from silently collapsing back to unstructured merged-project errors.

## Design direction

The diagnostics model is intended to support future:

- warnings and lints
- structured semantic expression spans
- multiple primary/secondary labels
- notes and suggestions
- machine-readable JSON diagnostics
- IDE/LSP diagnostics
- quick fixes
- go-to-definition/reference metadata
- macro / generated-source traces if those language features are added

The executable AST and Genix IR remain independent from CLI rendering concerns. Source metadata belongs to the frontend/source-map layer so interpreters and backends do not need to carry presentation-specific diagnostic state.

---

**Genix — by GenixBit**
