# Formatting Genix Source

Genix includes an official canonical formatter through the `gb` developer CLI.

```bash
gb fmt
```

The formatter is part of the pre-alpha toolchain and currently defines one canonical style rather than exposing per-project formatting options.

## Commands

Format the current Genix project:

```bash
gb fmt
```

Format another project:

```bash
gb fmt path/to/project
```

Format one `.gb` file:

```bash
gb fmt src/main.gb
```

Check formatting without modifying files:

```bash
gb fmt --check
gb fmt path/to/project --check
gb fmt src/main.gb --check
```

`--check` returns a non-zero process status when one or more files need formatting, making it suitable for CI.

## Project discovery

For a project target, `gb fmt` requires `genix.toml` and recursively formats:

```text
src/**/*.gb
tests/**/*.gb
```

It does not currently format `genix.toml`, generated native code, build artifacts, runtime C code, or files outside those project source/test trees.

## Canonical style

Genix currently uses four spaces for block indentation.

Before:

```gb
fn add(a:int,b:int)->int{return a+b;}
```

After:

```gb
fn add(a: int, b: int) -> int {
    return a + b;
}
```

Binary operators receive one space on each side:

```gb
let total: int = left + right;
let valid: bool = total >= 10 && total != 20;
```

Type annotations use one space after `:`:

```gb
let count: int = 10;
```

Function parameters use comma-space separation:

```gb
fn add(a: int, b: int) -> int {
    return a + b;
}
```

Generic-looking bootstrap types retain compact inner punctuation:

```gb
Option<string>
Result<string,string>
```

This means even compressed input such as:

```gb
let result:Result<string,string>=Ok("done");
```

is normalized to:

```gb
let result: Result<string,string> = Ok("done");
```

Blocks are expanded and indented consistently:

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

Top-level declarations are separated by a blank line.

## Test formatting

The formatter understands the `gb test` surface syntax even though test declarations are not part of normal application parsing.

Before:

```gb
test "addition works"{assert(2+2==4);pass();}
```

After:

```gb
test "addition works" {
    assert(2 + 2 == 4);
    pass();
}
```

This lets one formatter cover both ordinary source files and `tests/**/*.gb`.

## Comments

`gb fmt` preserves `//` comments.

Standalone comments remain standalone:

```gb
fn main() {
    // Explain why this value matters.
    let value: int = 10;
}
```

Trailing comments remain attached to the statement:

```gb
let value: int = 10; // retained by the formatter
```

The formatter also preserves string literal contents exactly. Whitespace inside:

```gb
"a   b"
```

is not rewritten.

## Idempotence

Canonical formatting is idempotent:

```bash
gb fmt
gb fmt
```

The second run should produce no source changes.

CI validates this property for standalone source as well as project source/test files.

## `--check` workflow

A common CI sequence is:

```bash
gb fmt --check
gb check
gb test
```

If formatting differs, `gb fmt --check` lists the affected files and fails without rewriting them.

Developers can fix the tree with:

```bash
gb fmt
```

## Architecture

The current formatter is deliberately a comment-preserving lexical formatter rather than an AST or IR pretty-printer.

```text
.gb source
    ↓
formatter tokenizer
    ↓
comment/string-preserving token stream
    ↓
canonical spacing + indentation printer
    ↓
.gb source
```

The distinction matters because the executable Genix AST intentionally does not retain comments. Formatting through the AST would therefore risk deleting source comments.

Genix IR is also intentionally not involved. Formatting is a source-level developer-tooling concern and must not depend on backend lowering.

## Safety properties

The formatter is designed to preserve:

- string literal contents
- `//` comments
- token order
- operators
- declarations and expressions
- test declarations and assertion syntax

The compiler CI formats intentionally compressed source and then verifies that the result still passes `gb check` and `gb test`.

## Current limitations

The pre-alpha formatter currently does not:

- wrap or reflow long lines to a configurable width
- reorder imports
- format `genix.toml`
- preserve arbitrary user-selected blank-line layouts
- expose `.genixfmt` or manifest-based style options
- format block comments, because Genix currently supports line comments only
- perform semantic rewrites or optimization

There is intentionally one canonical style for now. Configuration should only be introduced if real ecosystem needs justify divergence.

## Compatibility direction

Formatter output may still change during pre-alpha as syntax evolves. Once Genix reaches a language/tooling stability milestone, canonical formatting behavior should be versioned with the compiler so CI results remain reproducible.

---

**Genix Formatter — pre-alpha developer tooling**
