# Control Flow

Genix supports boolean expressions, conditional execution, loops, mutable variables, and lexical block scope.

> Status: pre-alpha. This document describes the behavior implemented by the current interpreter.

## Immutable and mutable variables

Variables declared with `let` are immutable:

```gb
let language = "Genix";
```

Variables that need reassignment must be declared with `mut`:

```gb
mut count = 0;
count = count + 1;
```

Attempting to assign to a `let` variable is rejected by the interpreter.

## Comparisons

Genix currently supports:

```text
==  !=  <  <=  >  >=
```

Example:

```gb
let age = 25;
print(age >= 18);
```

Ordering comparisons currently require numeric operands. Equality supports numbers, booleans, and strings. Integer/float numeric equality is supported across the two numeric representations.

## Boolean logic

Genix supports:

```text
&&  ||  !
```

`&&` and `||` use short-circuit evaluation.

```gb
let ready = true;
let enabled = true;

if ready && enabled {
    print("Ready");
}
```

Conditions must evaluate to `bool`; Genix does not currently use truthy/falsy coercion.

## if / else

```gb
fn main() {
    let age = 25;

    if age >= 18 {
        print("Adult");
    } else {
        print("Minor");
    }
}
```

Parentheses around the condition are not required.

## while

```gb
fn main() {
    mut count = 0;

    while count < 5 {
        print(count);
        count = count + 1;
    }
}
```

The condition is evaluated before every iteration and must return `bool`.

## Block scope

Every `{ ... }` block creates a lexical scope.

```gb
fn main() {
    let name = "outer";

    {
        let local = "inside";
        print(name);
        print(local);
    }

    // local is no longer available here.
}
```

Assignments search outward through active scopes, so a mutable outer variable can be updated from inside an `if`, `while`, or explicit block.

## Operator precedence

From lowest to highest precedence:

1. `||`
2. `&&`
3. `==`, `!=`
4. `<`, `<=`, `>`, `>=`
5. `+`, `-`
6. `*`, `/`
7. unary `!`, unary `-`
8. literals, variables, and parenthesized expressions

Use parentheses when you want to make grouping explicit.

---

**Genix — by GenixBit**
