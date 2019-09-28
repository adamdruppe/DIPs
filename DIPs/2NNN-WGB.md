# Shared Atomics

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0                                                               |
| Author:         | Walter Bright walter@digitalmars.com                            |
| Implementation: | https://github.com/dlang/dmd/pull/10209                         |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Reads and writes to data typed as `shared` are made atomic where the target CPU supports
atomic operations, and become erroneous when they cannot be done atomically.


## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

D has made an effective innovation by making shared types a first class type in the
language. Being able to distinguish the two is critical for developing robust multi-threaded
applications. But D stops short of altering the semantics of access to shared data,
making the default behavior subject to data races both obvious and hidden as the result of code
motion by optimizing compilers.

By making direct access to shared data not allowed, it will require the user to use `core.atomic`
and at least think about correctness of the code.

### Using core.atomic instead

This change will require using `core.atomic` or equivalent functions to read and write
to shared memory objects. It will prevent unintended, inadvertant non-use of atomic access.


### Limitations

This proposal does not guarantee code will be deadlock-free, nor does it obviate the need
for locks for transaction-race-free behavior. (A transaction is a sequence of operations
that must occur without another thread altering the shared data before the transaction is completed.)

## Description

Atomic reads perform an acquire operation, writes perform a release operation, and read-modify-write
performs an acquire, the modify, and then a release. The result is sequentially consistent ordering,
and each thread observes the same order of operations (total order).

The code generated would be equivalent to that from the core.atomic library for the supported operations.

There are no syntactical changes.

The supported operations are implementation defined for the target processor.
Supported operations are expected to be ones that are efficiently supported by the target
processor.

Inefficient operations will have to be programmed by the user with locks, and will thereby stand out in
source code preventing hidden performance surprises.

Whether a specific atomic operation is supported or not can be tested at compile time using
`__traits(compiles,...)`.


### Optimizations

Much like the C volatile type, may optimizations are not performed on atomic operations.
Any optimizations that would:

* cache the value of an atomic operation
* allocate an atomic variable in a register
* re-order atomic operations

Such optimizations include Common Subexpression Elimination, Copy Propagation, Loop Invariant
Removal, Dead Store Elimination, etc.

There are more code motion restrictions:

```
__gshared int g=0;
shared int x=0;

void thread1(){
    g=1; // non-atomic store
    x=1; // atomic store, release, happens after everything above
}

void thread2(){
    if(x==1){ // atomic load, acquire, happens before everything below
        assert(g==1); // non-atomic load
    }
}
```

I.e. code motion of `__gshared` cannot move across atomic operations. Surprisingly,
this applies to locals as well even though locals are not supposed to be visible
to other threads. This is because:

1. the address of a `__gshared` variable is taken, the type system does not distinguish
between that and the address of a local variable, so the optimizer must assume it is
`__gshared`.

2. the implementor of lock-free algorithms may wish to temporarily cast a shared variable
to unshared to get the expected non-atomic, but synchronized, behavior seen in the example above.
This would break if the compiler moved locals across atomic operations, and so such code
motion cannot be done.


### Risks

Will it work? Since it's just replacing calls to core.atomic with language support, and atomics
are well understood (at least by experts) in multiple languages, there should be little risk.
This is not pioneering new ground. The only innovation is support by the language
type system rather than library calls.


### Alternatives

Provide limited support for locked operations with operators, where the CPU supports it.
C++ provides such. This is controversial, as some believe it encourages incorrect coding
practices.


## Breaking Changes and Deprecations

All code that accesses shared memory objects will break.
This means there'll be a long deprecation cycle.

Code that uses `core.atomic` should continue to work correctly.

Code that is protected by locks will need to 'strip' the shared off of the head
of the type so that normal code generation can be performed on them.
This is performed via `cast()expression` or `cast(const)expression`.
Users will have to be careful not to let any references to those head-unshared memory
locations escape the locked code region, but that was true anyway prior to this change.


## Reference

[1] [Sequential Consistency](https://en.wikipedia.org/wiki/Sequential_consistency)
[2] [Data race](https://en.wikipedia.org/wiki/Race_condition#Software)

## Copyright & License

Copyright (c) 2019 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews

