# Argument Ownership and Function Calls


| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | XXXX                                                            |
| Review Count:   | 0                                                               |
| Author:         | Walter Bright                                                   |
| Implementation: |                                                                 |
| Status:         |                                                                 |


## Abstract

The language features proposed in DIP 25 and DIP 1000 greatly improved the memory safety of passing
references and pointers to functions by detecting if those
pointers escape a function's scope. Subsequently, a container
can safely pass an internal reference to a function if that
function doesn't allow the reference to escape.

But if a function is passed more than one reference to the same container,
one reference can render invalid the data referred to by the other reference(s).
This DIP aims to correct that problem. It's a natural progression after
DIP 25 and DIP 1000 which is needed to safely implement Reference Counting.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

Containers cannot be memory-safe if they cannot control memory-safe
access to their payloads. Containers cannot be efficient if they cannot
make available direct references to their payloads. Entreating the user
not to do certain things is unreliable and does not scale.

The simplest illustration of the problem:

```
struct S {
    byte* ptr;
    ref byte get() { return *ptr; }
}

void foo(ref S t, ref byte b) {
    free(t.ptr);    // frees memory referred to by b
    b = 4;          // corrupt memory access
}

void test() {
    S s;
    s.ptr = cast(byte*)malloc(1);
    foo(s, s.get());  // (*)
}
```
The same problem using scope pointers:
```
struct S {
    byte* ptr;
    byte* get() { return ptr; }
}

void foo(scope S* t, scope byte* pb) {
    free(t.ptr);    // frees memory referred to by pb
    *pb = 4;        // corrupt memory access
}

void test() {
    S s;
    s.ptr = cast(byte*)malloc(1);
    foo(&s, s.get());  // (*)
}
```

D currently has no defense against this problem, rendering checkably memory-safe
reference counting impossible. (Timon Gehr first pointed this out.)

## Prior Work

Rust avoids this problem with the following rule:

```
First, any borrow must last for a scope no greater than that of the owner.
Second, you may have one or the other of these two kinds of borrows, but not
both at the same time:

1. one or more references (&T) to a resource,
2. exactly one mutable reference (&mut T).
```
  https://doc.rust-lang.org/1.8.0/book/references-and-borrowing.html#the-rules

## Description

The solution hinges on the recognition that, in the example, two mutable references
to the same data are passed to the function `foo()`. Generalizing, whenever there are
multiple references to the same data and one of them is mutable, the mutable reference
can be used to invalidate the data the other references refer to, whether they are
`const` or not. Therefore, if more than one reference to the same data is passed to
a function, they must all be `const`.

This builds on the foundation established and tested by DIP 25 and DIP 1000, which track lifetimes
through function calls and their returns, by adding an additional check on data
already collected by the compiler semantics. The checks would only be enforced for `@safe` code.

### Syntax

This DIP proposes no syntax changes. It adds additional semantic checks on existing
constructs.

### Limitations

The proposed feature only checks expressions that are function calls. It does not perform interstatement
checking. It does not check non-scope pointers. It is not a complete borrowing/ownership
scheme, although it is an important step in that direction. For example, the checking
of scoped pointers can be defeated by using a temporary:

```
void test() {
    S s;
    s.ptr = cast(byte*)malloc(1);
    auto ps = &s;
    foo(ps, s.get());  // (*)
}
```
The only way to resolve this is by using [Reaching Definitions](#reference) data flow analysis, which
would be a fairly significant addition to the compiler. References, on the other hand, don't
need Reaching Definitions because they can only be initialized once and that is always
in scope.


## Breaking Changes and Deprecations

This will break existing code that passes to a function multiple mutable references to the same object.
It is unknown how prevalent a pattern this is. Breakage can be fixed either by
finding another way to pass the arguments or by marking the code as `@trusted`
or `@system`.


## Reference
Reaching Definitions: https://en.wikipedia.org/wiki/Reaching_definition


## Copyright & License

Copyright (c) 2019 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
