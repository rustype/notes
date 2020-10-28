# Foundations of Session Types and Behavioural Contracts

---

**Authors** — Hans Hüttel, Ivan Lanese, Vasco T. Vasconcelos, Luís Caires, Marco Carbone, Pierre-malo Deniélou, Dimitris Mostrous, Luca Padovani, António Ravara, Emilio Tuosto, Hugo Torres Vieira, Gianluigi Zavattaro\
**DOI** — [10.1145/2873052](https://doi.org/10.1145/2873052)\
**File** — [behavioral/session-types/2873052.pdf](https://github.com/rustype/bibliography/blob/main/behavioral/session-types/2873052.pdf)

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



---

**Notes**

- What is intensional and extensional aspects/information?

**References**

- <span id="1">Session types = intersection types + union types. (*Luca Padovani*)</span>

---