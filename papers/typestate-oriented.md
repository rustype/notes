# Typestate-Oriented Programming

---

**Authors** — Jonathan Aldrich, Joshua Sunshine, Darpan Saini, Zachary Sparks\
**DOI** — [10.1145/1639950.1640073](https://doi.org/10.1145/1639950.1640073)

---

The paper proposes the usage of objects as states,
allowing the type-system to track object states.
The introduction simply states how useful it is to do so.

## Context & Approach

The authors review a series of previous state-based approaches.

### State-Based Designs

Due to the lack of typestate support,
programmers are required to check for state during runtime,
using the traditional `if` clauses and other assertion mechanisms to ensure correctness.

While it is an extensible approach, it fails for larger coordination efforts.

### Typestate Checkers

Advances in typestate research focused on specifying the interface of a class in terms of states and
ensuring that clients only call functions that appropriate in a given state.

Typestate verification systems such as [[1](#1)] are based on techniques such as,
classifying references as not aliased or possibly aliased,
associating class states with invariants describing the class's fields and
frame-based approaches to support verification in the presence of inheritance.

Another possible approach [[2](#2), [3](#3)] is to specify state machine constraints on how an abstraction is used
and then use static analysis to check if they are followed.

### Benefits of a Language-Based Approach

The authors present a series of arguments supporting the language-based approach to typestates.

1. Incorporating typestate explicitly into a programming language encourages library writers and users to think in terms of states.
    This should help them design, document and reuse library components in a more effective manner.
2. Typestate adoption faces an entry barrier as it adds complexity on top of existing type systems,
    such as generics (e.g. Java) and type parameterization.
    Adding typestate directly to the language enables better reuse of mechanism between the type system and the state-tracking system, yielding a simpler and more understandable design.
3. Adding typestate to the language opens new opportunities for expressiveness.

### Prior Language-Based Approaches

The Actor model was one of the first programming models to treat states in a first class way.
An Actor can accept one of several messages,
in response it can perform the required tasks and then determine its next state.

Smalltalk introduced the `become` method,
which allows an object to exchange state and behavior with another object,
this mechanism can be used to model state changes in a first class fashion.

The concept of state is related to that of a role during interactions with an object.
A proposal was to allow objects to transition between roles as a form of state transition.

Another proposal, closely related to the object-oriented paradigm was the extension of class-based languages with explicit definitions of logical states, named modes, each with its own set of operations and corresponding implementations.

## Typestate-Oriented Programming

The paper describes the typestate-oriented programming paradigm,
discussing several topics related to the programming language [Plaid](https://plaid-lang.org).

### States

Plaid abstracts states as a class-like mechanism,
the difference resides in the fact that the state of an object my change during program execution.

An example from the original paper is:

```
state File {
    public final String filename;
}

state OpenFile extends File {
    private CFilePtr filePtr;
    public int read() { ... }
    public void close() [OpenFile>>ClosedFile] { ... }
}

state ClosedFile extends File {
    public void open() [ClosedFile>>OpenFile] { ... }
}
```

### Tracking State Changes

Methods can mutate an object state, if they do so, the `Before >> After` syntax is used.
If a method does not change the state of an object the used syntax is simply `State` as it is an abbreviation for `State>>State`.

This is clear in the methods `close` and `open` from the `OpenFile` and `ClosedFile`, respectively.

### Aliasing and Permissions

To deal with aliasing, Plaid adopted a permission-based system.

The simpler models are `unique` and `immutable`.
The former is unable to be aliased as indicated by the name,
the latter allows unlimited aliasing, since it is immutable it never changes.

There is also a `shared` aliasing state, which is much more complicated to keep track of.


---

**Notes**

The authors provide an example of a file when discussing typestate,
the file can be in two states `Open` and `Closed`.
The authors elaborate that in a traditional context the file would have a pointer to the file descriptor in the `Open` state, and in the `Closed` state, there is no need for such.
However, such approach raises an issue, how do we track memory usage as objects shrink and grow?
Should we just give an object the biggest possible size, like Rust's `enum`?

**References**

1. <a id="1">Typestates for Objects (*Robert Deline, Manuel Fahndrich*)</a>
2. <a id="2">Effective Typestate Verification in the Presence of Aliasing (*Stephen J. Fink, Eran Yahav, Nurit Dor, G. Ramalingam, Emmanuel Geay*)</a>
3. <a id="3">Collaborative Runtime Verification with Tracematches (*Eric Bodden, Laurie Hendren, Patrick Lam, Ondrej Lhotak, Nomair A. Naeem*)</a>

---

