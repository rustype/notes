# Channels as Objects in Concurrent Object-Oriented Programming

---

**Authors** — Joana Campos, Vasco T. Vasconcelos\
**DOI** — [10.4204/EPTCS.69.2](https://doi.org/10.4204/EPTCS.69.2)\
**File** — [behavioral/session-types/1110.4157v1.pdf](https://github.com/rustype/bibliography/blob/main/behavioral/session-types/1110.4157v1.pdf)

---

There are often semantic restrictions in the implementation of classes that impose particular sequences of legal method calls and aliasing restrictions.
These protocols are usually defined in informal documentation, being unable to be statically checked.

The authors present MOOL (Mini Object-Oriented Language) which apart from features such as concurrency,
allows programmers to specify usage protocols.
These protocols are formalized as usage types that specify available methods,
tests the client must perform on the returned values and existing aliasing restrictions.

## Objects

In MOOL, objects are governed by their type, where linear objects may evolve to shared ones,
the reverse is not possible as the number of references to a shared object is not tracked.

It is proposed that the programmer annotates objects to distinguish between linear (`lin`) and shared (`un`) objects.

- `lin` — describes the status of an object that can be referenced in exactly one thread.
- `un` — describes the status of an object that can be referenced in multiple threads.

## Approach

MOOL's approach is similar to [Modular Session Types](#2), defining a global usage specification of the available methods and extending it in two forms:

- Eliminating channels and replacing such system with a message passing mechanism in the form of method calls.
    These method calls implement a mechanism similar to the `synchronized` keyword from Java,
    in MOOL's case, the `sync` keyword.
- Expanding the type support beyond linear types,
    including shared types and treating them under a unified category.

It further expands usage specifications to separate methods and classes,
this implies that objects are non-uniform (able to dynamically change the set of available methods).

## MOOL

As previously noted, the MOOL language is object-oriented and defines usage types to constrain object usage.

In the case of a conventional class `C` containing methods `m1`, `m2` and `m3`,
the compiler will define `C`'s usage type as `usage * {m1 + m2 + m3}`,
meaning each method is always available.

For linear classes, the usage type declaration can be thought like `usage (lin .* ;)+ end;`,
consider the previous `C` class, if the declared methods are linear then a possible usage declaration would be `usage lin m1; lin m2; lin m3; end;`.

The `end` state is an abbreviation of `un {}`, which represents the empty set of available methods,
meaning the protocol has reached its final state.

A variant type is denoted by `< ... + ... >`, is indexed by the `true` and `false` values returned by the method to which the variant is bound.
A client should check the result of the call, if the result is `true` then the new state and available methods are found on the variant's right side, conversely, if the result is `false`, the new state and methods are found on the left side of the variant.

## Remarks

The MOOL language elevates session types to communications channels,
which in turn are treated as objects.

The paper takes inspiration from [Modular Session Types](#2),
the operations on communication channels are replaced by remote method invocations.

The present work also uses a standard synchronization mechanism to control concurrency,
instead of the well-known shared channel on which linear channels are created.
It also proposes a simple operational semantic and static type system,
the latter checks client code conformance to method call sequences and behaviorally constrained aliasing.

---

**Interesting References**

1. <span id="1">Regular types for active objects
    (*Oscar Nierstrasz*)</span>
2. <span id="2">Modular Session Types for Distributed Object-Oriented Programming
    (*Simon Gay, Vasco T. Vasconcelos, António Ravara, Nils Gesbert & Alexandre Z. Caldeira*)</span>

---