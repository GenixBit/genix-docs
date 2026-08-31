# Genix Documentation

Official documentation and evolving language specification for **Genix**, the `.gb` programming language by **GenixBit**.

> Status: pre-alpha. Language syntax, IR, runtime ABI, stdlib APIs, and compiler behavior may change before the first stable release.

## Current documentation

- `control-flow.md` — mutability, comparisons, boolean logic, `if` / `else`, `while`, and lexical block scope
- `functions-and-types.md` — functions, parameters, returns, explicit types, type inference, static checking, and numeric widening
- `error-handling.md` — `Option<T>`, `Result<T,string>`, `Some/None`, `Ok/Err`, exhaustive `match`, and `?` propagation
- `diagnostics.md` — compiler error codes, source spans, labels, help text, and diagnostics architecture
- `source-maps.md` — multi-file source identity, function/module mappings, primary spans, and secondary diagnostic labels
- `testing.md` — `gb test`, test discovery, `T0001` / `T0002` failures, exact assertion/fail spans, isolation, and current testing limitations
- `formatting.md` — `gb fmt`, canonical style, comment preservation, idempotence, project discovery, and `--check`
- `projects-and-modules.md` — `genix.toml`, multi-file projects, imports, module namespaces, and project checking
- `intermediate-representation.md` — typed Genix IR, lowering, explicit casts, backend contract, and `gb ir`
- `native-compilation.md` — native `gb build`, C11 backend, debug/release builds, and compiler requirements
- `runtime-abi.md` — external runtime integration, ABI v1, lifecycle, allocation, strings, output, and compatibility
- `standard-library.md` — stdlib discovery, compatibility, safe I/O, and standard APIs
- `native-intrinsics.md` — bootstrap host-service bridge for input, filesystem, environment, and process control

## Current project flow

```bash
gb new hello-genix
cd hello-genix

export GENIX_STDLIB=/path/to/genix-stdlib
export GENIX_RUNTIME=/path/to/genix-runtime

gb fmt --check
gb check
gb test
gb run
gb ir
gb build
./build/hello-genix
```

`gb new` creates `src/main.gb` plus a starter `tests/smoke.gb` test.

## Current compiler pipeline

```text
Genix application
    +
Genix standard library
    ↓
module loader + SourceMap
    ↓
lexer → parser → AST
    ↓
static type checker
    ↓
typed Genix IR
    ↓
C11 backend
    ↓
generated application
    +
Genix Runtime ABI v1
    ↓
native executable
```

`gb run` executes the checked AST through the Rust interpreter. `gb build` lowers checked code to typed Genix IR and links generated native code against `genix-runtime`.

Testing and formatting are intentionally source/developer-tooling paths rather than application backend concerns.

```text
tests/*.gb                 src/**/*.gb + tests/**/*.gb
    ↓                                   ↓
Genix test frontend                 gb fmt
    ↓                                   ↓
parser + AST + checker       lexical canonical formatter
    ↓
fresh interpreter per test
    ↓
T0001 / T0002 diagnostics
```

Test declarations and formatter metadata do not enter normal Genix IR/native application builds.

## Testing

Project tests live in `tests/*.gb` and use named test blocks:

```gb
test "arithmetic works" {
    assert(2 + 2 == 4);
}
```

Run them with:

```bash
gb test
gb test path/to/project
gb test path/to/test.gb
```

The runner supports `assert(condition)`, `fail(message)`, and `pass()`, type-checks the complete suite, runs each test with a fresh interpreter instance, prints pass/fail totals, and returns a non-zero status when any test fails.

Failed assertions use `T0001` and point to the original assertion expression. Explicit `fail(message)` uses `T0002`, preserves the message, and points to the original failure site. Genuine interpreter/runtime errors remain runtime errors and are not converted into assertion failures.

See `testing.md` for the isolation model, failure-channel architecture, and current pre-alpha limitations.

## Formatting

Genix now has one canonical source formatter:

```bash
gb fmt
gb fmt src/main.gb
gb fmt --check
```

For project targets it recursively formats `src/**/*.gb` and `tests/**/*.gb`. It uses four-space indentation, canonical operator/type/function spacing, compact `Option<T>` / `Result<T,string>` punctuation, and preserves string contents plus `//` comments.

`gb fmt --check` performs no writes and returns a non-zero status if source needs formatting, making it suitable for CI. Formatter output is idempotent and is validated by compiler CI before the formatted source is passed back through `gb check` and `gb test`.

See `formatting.md` for the complete canonical style and architecture.

## Compiler diagnostics and source maps

Direct source-file commands render coded, source-aware diagnostics:

```text
error[E0201]: initializer for 'age' expected int, found string
 --> src/main.gb:2:20
   |
 2 |     let age: int = "twenty";
   |                    ^^^^^^^^ type mismatch
  = help: change the expression or annotation so the types are compatible
```

Current compiler-error families are:

```text
E000x  lexer
E010x  parser / syntax
E020x  static type checking
```

Test failures use the separate `T000x` family.

Project/module loading maintains a `SourceMap` that records original file text plus canonical function/module ownership. Imported-module lexer/parser errors therefore retain their real filenames, and semantic errors can be mapped back to the original module after the compiler merges module functions for checking.

Diagnostics also support secondary locations:

```text
 --> src/math.gb:2:22
 ::: src/main.gb:4:11
```

The primary location identifies the failing source. Related `:::` locations can show where a module was referenced or where a called function was defined.

The executable AST and Genix IR intentionally remain independent from diagnostic rendering metadata. See `source-maps.md` for the source-map contract, secondary-label model, and current semantic-adapter limitations.

## Typed error handling

Genix supports primitive-payload `Option` and `Result` values:

```gb
fn load(path: string) -> Result<string,string> {
    let text: string = fs.try_read_text(path)?;
    return Ok(text);
}
```

Exhaustive matching is required:

```gb
match result {
    Ok(value) => {
        print(value);
    }
    Err(error) => {
        print(error);
    }
}
```

Current safe host-facing APIs include:

```text
process.env_option(name) -> Option<string>
fs.try_read_text(path) -> Result<string,string>
fs.try_write_text(path, text) -> Result<bool,string>
```

## Standard library and host services

Current standard modules include:

```text
io
fs
process
math
string
```

Portable APIs remain ordinary `.gb` code. Foundational OS-facing calls use the bootstrap host intrinsic bridge so the same public Genix API works in both interpreter and native modes.

The compiler validates stdlib compatibility metadata against language version `0.0.1` and runtime ABI `1`.

## Genix IR

The IR records resolved function names, typed variables and expressions, structured control flow, explicit safe `int → float` casts, Option/Result constructors, matches, and Result propagation.

```bash
gb ir
gb ir path/to/project
```

## Roadmap

Documentation will continue to cover:

- Getting started and language tour
- Full grammar and generalized type system
- User-defined enums and generic types
- Modules and packages
- Structured semantic diagnostics with checker-native source spans
- Expected/actual assertion value reporting, filtering, and native test execution
- Genix IR optimization
- Ownership and memory safety
- Stable native FFI
- Runtime/platform APIs
- Standard library reference
- Target triples and cross-compilation
- LLVM and WebAssembly backends
- Concurrency and async
- Language server and editor tooling
- Language specification and compatibility policy

## Hello, Genix

```gb
fn main() {
    print("Hello from Genix!");
}
```

## Core identity

| Item | Value |
|---|---|
| Language | Genix |
| Source extension | `.gb` |
| Primary CLI | `gb` |
| Compiler identity | `gbc` |
| Intermediate representation | Genix IR |
| Runtime | Genix Runtime ABI |
| Standard library | Genix Stdlib |
| Creator | GenixBit |

## Documentation policy

Experimental proposals are marked as experimental. Behavior should only be described as stable after corresponding compiler/runtime/stdlib tests and an accepted specification exist.

---

**Genix — by GenixBit**
