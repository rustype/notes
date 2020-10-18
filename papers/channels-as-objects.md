# Channels as Objects in Concurrent Object-Oriented Programming

---

**Authors** — Joana Campos, Vasco T. Vasconcelos\
**DOI** — 10.4204/EPTCS.69.2

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

MOOL's approach is similar to [2], defining a global usage specification of the available methods.
It extends [2] in two forms:

- Eliminating channels and replacing such system with a message passing mechanism in the form of method calls.
  These method calls implement a mechanism similar to the `synchronized` keyword from Java,
  in MOOL's case, the `sync` keyword.
- Expanding the type support beyond linear types,
  including shared types and treating them under a unified category.

It further expands usage specifications to separate methods and classes,
this implies that object become non-uniform (able to dynamically change the set of available methods).


---

**Interesting References**

1. Regular types for active objects. (*Oscar Nierstrasz*)
2. Modular Session Types for Distributed Object-Oriented Programming.
  (*Simon Gay, Vasco T. Vasconcelos, António Ravara, Nils Gesbert & Alexandre Z. Caldeira*)

---