# Functions and Static Types

Genix supports user-defined functions, typed parameters, typed return values, function calls, `return`, and static type checking before execution.

> Status: pre-alpha. This document describes the behavior implemented in the current `genix-lang` interpreter.

## Function declarations

```gb
fn add(a: int, b: int) -> int {
    return a + b;
}
```

A function declaration contains:

- `fn`
- Function name
- Parameter list
- A type for each parameter
- Optional `->` return type
- Function body

## Void functions

A function without a declared return type is a void function:

```gb
fn greet(name: string) {
    print("Hello " + name);
}
```

It may be called as a statement:

```gb
greet("GenixBit");
```

## Return values

```gb
fn square(value: int) -> int {
    return value * value;
}
```

Functions with non-void return types must guarantee a return value. The current checker recognizes direct returns, returns inside blocks, and `if`/`else` branches where both branches return.

```gb
fn classify(value: int) -> string {
    if value >= 10 {
        return "large";
    } else {
        return "small";
    }
}
```

## Built-in value types

The current core type system provides:

| Type | Meaning |
|---|---|
| `int` | Signed integer |
| `float` | Floating-point number |
| `string` | UTF-8 string value |
| `bool` | `true` or `false` |

`void` is used for functions that do not return a value.

## Variable type inference

A variable may infer its type from its initializer:

```gb
let age = 25;
let language = "Genix";
let ready = true;
```

## Explicit variable types

Variables may also specify their type:

```gb
let age: int = 25;
let price: float = 99.5;
let name: string = "Genix";
let ready: bool = true;
```

Mutable typed variables use `mut`:

```gb
mut count: int = 0;
count = count + 1;
```

## Static type errors

Genix checks types before executing the program.

Invalid:

```gb
let age: int = "twenty";
```

Diagnostic:

```text
Genix error: type error: initializer for 'age' expected int, found string
```

Function arguments are checked too:

```gb
fn add(a: int, b: int) -> int {
    return a + b;
}

fn main() {
    let result = add(1, "two");
}
```

The second argument is rejected because `string` is not compatible with `int`.

## Numeric widening

Genix currently permits safe widening from `int` to `float`:

```gb
fn double(value: float) -> float {
    return value * 2.0;
}

fn main() {
    let result: float = double(3);
    print(result);
}
```

The integer `3` is accepted where a `float` is expected and is converted to `3.0` by the runtime.

The reverse conversion (`float` to `int`) is not implicit.

## Function calls in expressions

Functions returning a value can be used inside expressions:

```gb
fn add(a: int, b: int) -> int {
    return a + b;
}

fn main() {
    let result: int = add(10, 20) * 2;
    print(result);
}
```

Void functions cannot be used where a value is required.

## Entry point

A Genix program must currently define:

```gb
fn main() {
}
```

In Genix v0.1, `main` cannot accept parameters or declare a return value.

## Current limitations

The function/type system is intentionally small while the language semantics stabilize. Not yet implemented:

- Generic types
- Function overloading
- Named/default arguments
- Closures
- Lambda expressions
- User-defined structs/classes
- Interfaces/traits
- Nullable/optional types
- Cross-file function imports

These will be considered in later language milestones.

---

**Genix — by GenixBit**
