# Genix Standard Library

The official standard library is maintained in `genix-stdlib` and loaded by the compiler as `.gb` modules.

> Status: pre-alpha / stdlib 0.0.1.

## Setup

```bash
export GENIX_STDLIB=/path/to/genix-stdlib
export GENIX_RUNTIME=/path/to/genix-runtime
```

## Import resolution

For `import math;`, Genix searches:

1. the project-local module beside the entry source;
2. `GENIX_STDLIB/modules/math.gb`.

Project-local modules intentionally take precedence.

## Compatibility

```text
GENIX_STDLIB_VERSION=0.0.1
GENIX_LANGUAGE_VERSION=0.0.1
GENIX_RUNTIME_ABI=1
```

The compiler validates this metadata before loading an official module.

## `io`

```text
io.println(text: string)
io.print_int(value: int)
io.print_float(value: float)
io.print_bool(value: bool)
io.input(prompt: string) -> string
```

```gb
import io;

fn main() {
    let name: string = io.input("Your name: ");
    io.println("Hello " + name);
}
```

## `fs`

```text
fs.read_text(path: string) -> string
fs.write_text(path: string, text: string)
```

```gb
import fs;

fn main() {
    fs.write_text("hello.txt", "Hello Genix");
    print(fs.read_text("hello.txt"));
}
```

Native filesystem failures currently panic through the runtime. Interpreter failures surface as execution errors. Structured `Result` behavior is planned.

## `process`

```text
process.env(name: string) -> string
process.exit(code: int)
```

A missing environment variable currently produces `""` until Genix has an `Option` type.

## `math`

```text
math.abs_int(value: int) -> int
math.abs_float(value: float) -> float
math.min_int(a: int, b: int) -> int
math.max_int(a: int, b: int) -> int
math.clamp_int(value: int, minimum: int, maximum: int) -> int
math.square(value: float) -> float
```

## `string`

```text
string.concat(left: string, right: string) -> string
string.equals(left: string, right: string) -> bool
string.not_equals(left: string, right: string) -> bool
```

## Architecture

```text
Application
    ↓
Genix stdlib
    ↓
portable `.gb` code
    +
bootstrap host intrinsics
    ↓
Genix Runtime ABI
    ↓
Operating system
```

The compiler currently recognizes these canonical stdlib functions as host intrinsics:

```text
io.input
fs.read_text
fs.write_text
process.env
process.exit
```

The interpreter performs equivalent Rust host operations; native builds lower them to runtime ABI functions. See `native-intrinsics.md` for the contract.

## Planned modules

- `path`
- `time`
- `collections`
- `json`
- `net`
- `http`
- `crypto`
- `concurrent`
- `test`

APIs remain experimental until covered by compiler, runtime, and stdlib compatibility tests.
