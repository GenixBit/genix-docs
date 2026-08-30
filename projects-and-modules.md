# Genix Projects and Modules

Genix projects use a `genix.toml` manifest and `.gb` files inside a source directory.

> Status: pre-alpha. The first module system is intentionally small and may change.

## Create a project

```bash
gb new hello-genix
cd hello-genix
gb run
```

Generated layout:

```text
hello-genix/
├── genix.toml
└── src/
    └── main.gb
```

## Project manifest

```toml
[project]
name = "hello-genix"
version = "0.1.0"
entry = "src/main.gb"
```

Current fields:

- `name` — project name
- `version` — project version
- `entry` — `.gb` entry file

The entry path must remain inside the project directory and must point to a `.gb` file.

## Importing a module

Create a sibling source file:

```text
src/
├── main.gb
└── math.gb
```

`math.gb`:

```gb
fn add(a: int, b: int) -> int {
    return a + b;
}

fn twice(value: int) -> int {
    return add(value, value);
}
```

`main.gb`:

```gb
import math;

fn main() {
    let answer: int = math.twice(21);
    print(answer);
}
```

The import:

```gb
import math;
```

maps to:

```text
src/math.gb
```

Functions exported by that file are called through the module namespace:

```gb
math.add(10, 20)
math.twice(21)
```

Functions inside the module may call each other without the namespace prefix. The project loader resolves those internal calls to the module namespace before static type checking.

## Commands

Run the current project:

```bash
gb run
```

Run another project directory:

```bash
gb run path/to/project
```

Check the current project:

```bash
gb check
```

Check another project:

```bash
gb check path/to/project
```

Create the current frontend build artifact:

```bash
gb build
```

This writes:

```text
build/genix.frontend
```

The artifact records project metadata and confirms that module loading, parsing, and static type checking succeeded. It is **not a native executable**.

## Current limitations

The first module-system version has deliberate limits:

- Imports are declared in the project entry file.
- Imported modules are sibling `.gb` files of the entry file.
- Nested imports inside imported modules are not supported yet.
- Modules cannot define `fn main()`.
- Package/registry imports are not implemented yet.

These constraints keep module resolution deterministic while the compiler frontend matures.

## Next module/tooling work

Planned extensions include:

- Nested module imports
- Directory modules
- Explicit exports/privacy
- Package dependencies
- Lockfiles
- Standard-library module resolution
- Better module-cycle diagnostics

---

**Genix — by GenixBit**
