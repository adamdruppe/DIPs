# Add `throw` as Function Attribute

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@digitalmars.com                            |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Add `throw` attribute as the inverse of the `nothrow` attribute.


## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

Currently, functions default to possibly throwing an exception. This can be turned
off by applying the `nothrow` attribute. But once `nothrow` is used, such as at module
scope, all functions in its scope are affected, there's no escape.

This is why aggregates do not "inherit" the `nothrow` attribute from the outer scope.
But if the entire module needs to be `nothrow`, it has to be applied not only at module
scope but in the definition of each aggregate:

```
void bar(); // may throw

struct S1 {
    nothrow void foo() { bar(); } // error, bar() throws
}

nothrow {
    struct S2 {
        void foo() { bar(); } // no error
    }
}
```

The problem is that exceptions are not cost-free, even in code that never throws. Exceptions
should therefore be opt-in, not opt-out. Although this DIP does not propose making exceptions
opt-in, adding the `throw` attribute is a key requirement for it. It also sits well as documentation
that yes, a function indeed can throw.


## Prior Work

The `@safe`, `@trusted` and `@system` attributes can override each other.


## Description

Add `throw` to [Attribute](https://dlang.org/spec/attribute.html#Attribute).
Add `throw` to [FunctionAttribute](https://dlang.org/spec/function.html#FunctionAttribute).
This produces an ambiguity with the start of a
[ThrowStatement](https://dlang.org/spec/statement.html#throw-statement)
but can be disambiguated by looking ahead to see if an Expression follows or a Declaration.

The attribute applies only to function and delegate types, for other types it is ignored.
It means that the function may throw an Exception.

```
void bar() throw;

struct S1 {
    nothrow void foo() { bar(); } // error, bar() throws
}

struct S2 {
    void foo() { bar(); } // no error
}
```

`throw` and `nothrow` cannot be combined, but can override one in an enclosing scope:

```
void abc() throw throw;   // Error
void bar() throw nothrow; // Error

nothrow:
    foo() throw; // Ok

throw:
    def() throw; // Ok
```


## Breaking Changes and Deprecations

None


## Reference

Herb Sutter notes in
[De-fragmenting C++: Making exceptions more affordable and usable](https://www.youtube.com/watch?v=os7cqJ5qlzo)
that half of C++ users either prohibit exceptions wholly or partially in their C++ code.


## Copyright & License
Copyright (c) 2019 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
