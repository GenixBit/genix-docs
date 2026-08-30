# Error Handling

Genix provides typed absence and recoverable-error handling through `Option<T>` and `Result<T, string>`.

> Status: pre-alpha / Genix 0.0.1. The current implementation intentionally supports primitive payload types first. The syntax is designed to grow into a more general enum/generic system later.

## Option

Use `Option<T>` when a value may be absent.

```gb
fn find_name() -> Option<string> {
    return Some("Genix");
}
```

An absent value is written as `None`:

```gb
fn find_name() -> Option<string> {
    return None;
}
```

Current supported payloads are:

```text
Option<int>
Option<float>
Option<bool>
Option<string>
```

## Result

Use `Result<T, string>` for recoverable failures.

```gb
fn load() -> Result<string,string> {
    return Ok("data");
}
```

Errors are represented with `Err(...)`:

```gb
fn load() -> Result<string,string> {
    return Err("could not load data");
}
```

Current successful payloads are:

```text
Result<int,string>
Result<float,string>
Result<bool,string>
Result<string,string>
```

The error type is currently `string`. Arbitrary error types and nested generic payloads are planned for a later type-system revision.

## Exhaustive match

`Option` and `Result` values are unpacked with `match`.

```gb
let value: Option<string> = Some("Genix");

match value {
    Some(name) => {
        print(name);
    }
    None => {
        print("missing");
    }
}
```

Result matching:

```gb
let result: Result<string,string> = load();

match result {
    Ok(text) => {
        print(text);
    }
    Err(error) => {
        print(error);
    }
}
```

The compiler requires both variants. A non-exhaustive `Option` or `Result` match is a type error.

## Error propagation with `?`

A function returning `Result<T,string>` can propagate another Result error with `?`.

```gb
fn load_config() -> Result<string,string> {
    let text: string = fs.try_read_text("config.txt")?;
    return Ok(text);
}
```

Conceptually:

```text
Result is Ok(value)  → unwrap value and continue
Result is Err(error) → return Err(error) immediately
```

In the current bootstrap implementation, `?` must be the complete value of a variable initializer, assignment, or call expression statement. Nested forms such as `Ok(load()?)` are intentionally deferred until the IR control-flow lowering is generalized.

`?` cannot be used from a function that does not itself return `Result<...,string>`.

## Safe standard-library APIs

Preferred recoverable filesystem APIs:

```gb
fs.try_read_text(path: string) -> Result<string,string>
fs.try_write_text(path: string, text: string) -> Result<bool,string>
```

Preferred optional environment lookup:

```gb
process.env_option(name: string) -> Option<string>
```

Example:

```gb
import fs;
import process;

fn save_and_load(path: string) -> Result<string,string> {
    let written: bool = fs.try_write_text(path, "Genix safe IO")?;
    let text: string = fs.try_read_text(path)?;
    return Ok(text);
}

fn main() {
    let home: Option<string> = process.env_option("HOME");

    match home {
        Some(value) => {
            print(value);
        }
        None => {
            print("HOME is not set");
        }
    }

    let result: Result<string,string> = save_and_load("data.txt");

    match result {
        Ok(text) => {
            print(text);
        }
        Err(error) => {
            print(error);
        }
    }
}
```

## Runtime representation

The current C11 native backend lowers primitive Option and Result types to tagged runtime ABI structures such as `GbOptionString` and `GbResultString`.

The representation is an implementation detail and may evolve before Genix reaches a stable ABI.

## Future work

Planned generalizations include:

- user-defined enums
- arbitrary generic payload types
- typed error objects beyond `string`
- nested `Option` / `Result`
- expression-valued `match`
- broader `?` placement
- richer diagnostic source spans
