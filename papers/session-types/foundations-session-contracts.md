# Foundations of Session Types and Behavioural Contracts

---

**Authors** — Hans Hüttel, Ivan Lanese, Vasco T. Vasconcelos, Luís Caires, Marco Carbone, Pierre-malo Deniélou, Dimitris Mostrous, Luca Padovani, António Ravara, Emilio Tuosto, Hugo Torres Vieira, Gianluigi Zavattaro

**DOI** — [10.1145/2873052](https://doi.org/10.1145/2873052)

**File** — [behavioral/session-types/2873052.pdf](https://github.com/rustype/bibliography/blob/main/behavioral/session-types/2873052.pdf)

---

Work on behavioural appeared in the context of type systems which capture properties of computations in process calculi.
Behavioural types are compositional in the sense that the type of a composite program depends on the types of its immediate constituents.

## Approaches to Behavioural Types

### Intersection Types

Intersection types introduce the type constructor `&`,
an identity will have type `T1 & T2` if it has both type `T1` and `T2`.
This enables the typing of a program which exhibits behavior corresponding to `T1` and `T2`,
further enabling a notion of ad hoc polymorphism.

#### Union Types

Union types are considered to be the dual of intersection types,
these are used to express uncertainty as to which type the entity has.

### Typestates

In the typestate approach, the type of an entity depends on the operations that are permitted for the entity, when at a particular state.
Each type has associated with it a set of typestates, partially ordered;
operations on entities of the type are correct if the resulting values are of a typestate reachable by a typestate transition (following order).

Typestates, are therefore, akin to finite-state machines,
and a language equipped with a static type system based on them can check at compile time if all possible sequences of operations are valid with respect to a correct use of the application.

### Types and Effects

A type and effect system makes it possible to statically describe intensional aspects of a computation alongside the extensional information that is captured by usual notions of type.
Types describe what an expression will compute, while effects describe how an expression will compute.

### Types for Nonuniform Objects

In the context of OOP, class types as static interface types do not cope with the notion of nonuniform method availability: In an object, each of its methods can be enabled or disabled according to its internal state.

Nonuniform objects are those that may dynamically change behavior,
and a typing discipline for ensuring the absence of "message-not-understood" errors will need to take this dynamic behavior into consideration.

A possible problem to this approach are orphan messages, these may fail to be accepted by any actor in the system due to the dynamic changes.
There has been some research on the topic, having been defined both a calculus and a type system for these kind of systems.

### Processes as Types

A possible approach which originates on type and effect systems,
is that of considering processes as types.
Here types are processes that are sound abstractions of the behavior of programs, and an analysis of the type thus becomes an analysis of the behavior of the program.

Similarly, session types can be seen as processes abstracting the protocol followed by programs.

### Interface Automata

Interface automata constitute an automata-based approach to behavioral types for specifying extensional program properties.

Interface automata are now used to specify interfaces of components, and a refinement relation is used to compare abstract and concrete interface specifications.
In much of this work the focus is not on establishing explicit typing rules for an underlying programming language but instead on defining notions of conformance, compatibility, and composition.

Interface automata are similar in aim to session types and behavioral contracts, but their automata-based presentation makes the two approaches differ technically.

## Binary Sessions

### IO Types and Linear Types for the Pi-Calculus

The simplest form of pi-calculus only keeps track of the number of arguments a channel may carry, this prevents arity mismatches,
where the number of channels sent by a client differs from the number expected by a server.
Types are tuples, of the form `T1, ..., Tn` describing communication channels able of carrying `n (where n >= 0)` channels of types `T1, ..., Tn`.

This type system can be refined, a possible one is the inclusion of polarity for IO types,
establishing that one may one be used to for input and the other for output.

Further refinements may be added, such the inclusion of ideas from linear logic, controlling the number a hypothesis can be used in a proof.

### Binary Session Types

A binary session type describes a protocol as seen from the point of view of one of the two participants.

A lot of what this section explains is present in [[session-types]].

Session types may allow for messages to carry other session types,
this phenomenon is called delegation and is considered the norm in session type systems.

### Binary Contracts

Contracts take an approach that differs from session types by using algebra-like languages or labelled transition systems
for describing abstractions of the communication behavior of programs.

## Multiparty Sessions

### Global and Local Types

Multiparty sessions types provide for global descriptions of interactive behavior.
Under this paradigm, a software architect prepares a global view of all the message exchanges that take place,
instead of separately defining the behavior of each individual channel endpoint.

The local behavior of each endpoint can be mechanically obtained from the global description by applying a projection operation.
A global description is therefore a "formal blueprint" of how a communicating system should behave and it provided a concise specification of how messages flow within the system.

---

**Notes**

- What is intensional and extensional aspects/information?

**References**

- <span id="1">Session types = intersection types + union types. (*Luca Padovani*)</span>

---

[//begin]: # "Autogenerated link references for markdown compatibility"
[session-types]: ../../notes/session-types.md "Session Types"
[//end]: # "Autogenerated link references"