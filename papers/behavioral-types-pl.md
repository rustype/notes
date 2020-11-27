# Behavioral Types in Programming Languages

---

**Authors** — Davide Ancona, Viviana Bono, Mario Bravetti, Joana Campos, Giuseppe Castagna, Pierre-Malo Deniélou, Simon J. Gay, Nils Gesbert, Elena Giachino, Raymond Hu, Einar Broch Johnsen, Francisco Martins, Viviana Mascardi, Fabrizio Montesi, Rumyana Neykova, Nicholas Ng, Luca Padovani, Vasco T. Vasconcelos, Nobuko Yoshida

**DOI** — [10.1561/2500000031](https://doi.org/10.1561/2500000031)

**File** — [biblio/behavioral/ancona2016.pdf](https://github.com/rustype/bibliography/blob/main/behavioral/ancona2016.pdf)

---

## Introduction

The introduction provides an overview on behavioral types and its descendants,
such as session types and typestates.

The paper reviews a series of subtopics about behavioral types and their implementation in programming languages.
I focus on the functional and object-oriented paradigms, while also reviewing work done on Singularity OS.

## Object-Oriented Languages

The research of behavioral types on OOP languages takes on two main lines, typestates and session types.
The former focus on constraints such as the method call order on an object,
objects using typestate disciplines can be seen as an automaton.
The latter can be seen as a special case of typestate in which ordered operations are either sends or receives in a communication channel.

### Behavioral Types in Java-like Languages

This section reviews existing work on Java extensions and Java-like languages supporting behavioral types.
There are features which are shared by all of them:
- Only objects of some specific classes are subject to the behavioral type system.
- Aliasing is disallowed for such objects.
- Behavioral type-checking can in principle be implemented as a first pass before the file is passed on to the Java compiler.
  - Syntactic extensions are either translated or erased after this pass.

The paper reviews work done on SessionJ, Mungo and MOOL. It also provides a quick overview over Yak, however, it is dismissed in favor of Mungo.
SessionJ and Mungo are extensions to the Java language, while MOOL is a standalone language.

#### SessionJ

SessionJ is the biggest project, having been developed over the years,
it is based on MOOSE, being adapted to Java.
In SJ, session channels are objects of one specific class and its usages is strictly controlled whereas objects from other classes are treated as in plain Java.
A communication channel endpoint is an object of class `SJSocket`, implemented on top of a TCP socket.

In SessionJ, a session-type channel cannot escape the method in which it was created except by being delegated.
Delegation is implemented transparently: the `SJSocket` object embodying the session-typed channel is passed as argument to the `send` method of another one.
It is also possible for a session channel to be passed as an argument to a regular method call,
the remainder of the session will be delegated to another local object rather than to another site.
A method which expects a channel endpoint as argument declares the expected session type as its argument type (rather than `SJSocket`).

#### Yak

Yak allows adding a usage protocol to any class, however, it has no specific construct for channels or concurrency.
It is very similar to Mungo, although the syntax is different, its main feature is exception handling in the protocols.

#### Mungo

Mungo tries to provide middle ground, seeking the integration of both approaches:
all classes can have usage protocols and communication channels are objects of one specific class.
Mungo aims for modularity, meaning that the session implementation on a communication channel can be separated into several methods,
each of which, expected the channel to be in a given state and leaving the channel in another one.

It presents a clean incorporation of session types in a Java-like language,
where sessions control the order in which methods are called, but also the choices imposed on clients by virtue of values returned by methods.
Type checking is strictly static and sessions do not exist in the semantics, implementation can thus be done as a verification pass before compilation.

Channels are session types are not native to Mungo, instead,
a channel session type is translated into an object protocol that specifies the allowed sequence of calls of `send`/`receive` methods on an object that encapsulates a channel endpoint.
This can be seen as the unification of the more general concept of typestate.

The usage channel protocol for the channel class is not fixed, that is,
instances of that class are created by initializing a communication session,
and their initial usage protocol is determined from the associated session type.

#### MOOL

MOOL provides usage protocols and concurrency, however, it does not provide explicit channels.
Message passing is done through regular method calls.
MOOL also deals with linear types as well as shared ones, treating them in a unified framework.

Classes are annotated with a usage descriptor that structures method invocation,
enhanced by `lin`/`un` aliasing qualifiers. A single category for objects is defined,
in which an object may evolve from linear status into an unrestricted (or shared) one.

### Typestate

Whereas the type of an object specifies all operations that can be performed on the object,
typestates identify subsets of these operations that can be performed on the object in particular abstract states.

A typestate pre-condition must hold for an operation to be applicable,
and a typestate post-condition reflects the possible typestates after the operation has been applied.

An important observation, made by the original authors of typestates,
is that static checking typestates in the presence of unrestricted aliasing and concurrency is impossible.
However, it is possible to apply static checking for controlled concurrency and dynamic process creation.

#### Vault

Vault is a typestate-checking system for an extension of C.

The alias control is based on an association between *keys* and *tracked resources*.
A resource with a typestate specification must be tracked, which means that the type system attaches a unique key to it.

Aliases for the resource can be created freely, but applying a typestate-changing operation to a resource also requires the correct key;
the alias control problem is thus transferred to the keys.

A system called *adoption and focus* allows temporary aliases to be created within a local scope and checks that they are destroyed by the end of the scope.
Keys exist only in the static type system, and are not required at runtime.

#### Fugue

Fugue is a modular verification system for specifying and statically checking typestate properties for .NET programs.
It adopts the original concept of typestates to object-oriented programs.

In Fugue a typestate is an abstraction over the concrete state of an object,
specified by a predicate over the fields of the object.
This approach faces two main challenges:
- The actual definition of a typestate depends on the subclass relation.
- Typestates must be uniform.

To solve such challenges, Fugue makes use of frame typestates,
these define the property corresponding to a typestate for each subclass,
and by sliding methods, which ensure that subclasses override methods of superclasses which change the typestate,
such that the change also applies in the subclass.

To address aliasing, Fugue uses the adoption and focus model, distinguishing two modes for object references: `NotAliased` and `MayBeAliased`.
`NotAliased` references may become `MayBeAliased`, the `MayBeAliased` typestate is final.

#### Plural

Plural is a Java extension, implemented as an Eclipse plugin.

It uses fractional permissions for alias control,
allowing a single object reference, with a certain access permission,
to be split into fractions which are later recombined to restore the full permission.

Plural also supports concurrency by means of synchronization and atomic blocks,
and allows typestate definitions to be defined over collections of interacting objects.

#### Plaid

In conjunction with the introduction of typestate as a language paradigm,
Plaid was introduced, materializing it.

In Plaid, typestates are similar to classes, however different typestates may have the same values for the fields of the concrete state,
but the fields of different typestates need not be the same.

## Functional Languages

The integration of sessions and session types in functional languages poses two main challenges.

The first one concerns the fact that, by their own definition,
session types describe entities (channel endpoints) whose type may change after each usage,
this is at odds with the conventional notion of type used in functional languages,
which is meant to describe the nature of values.

The second challenge concerns the fact that the call-by-name evaluation strategy adopted in lazy functional languages such as Haskell makes it difficult to predict the order in which expressions are evaluated.
This contrasts with the need to perform IO operations over channel endpoints in an order which is precisely determined by a session type.

The first challenge has been addressed in three different ways:

- Drawing inspiration from effect systems
- Using explicit continuation for channels
- Defining a suitable monad for session communications

Monads also prove to be useful when addressing the second challenge,
the same way the `IO` monad allows the structuring of Haskell programs doing IO.

### Effects for Session Type Checking

[Vasconcelos et al.](#ml-effects) have investigated the integration of sessions into an ML-like language by devising a session type system that keeps track of the effect of a function on the channel it uses.
Like traditional effect systems, the session type system decorates arrow types with information on the effect of a function.
In this case, however, the decorations are solely meant to capture the change in the type of channels passed to the function, when the function is applied.

### Sessions and Explicit Continuations

The idea of continuations is that a function that takes a channel as argument consumes the channel and produces a continuation of the same channel with a possibly different type,
this idea is based on the observation that channels are linear resources that must be used exactly once.

Partial application imposes a problem, since traditional type systems do not recognize linear types in such context,
enabling the values to be reused.
To address this problem, the authors create a new arrow notation for linear types,
enabling the type system to detect linear resources.

### Monadic Approaches to Session Type Checking

Support for session types in Haskell has been investigated by several existing works.
When adding support for session interaction primitives in a lazy functional language special care is required,
operations must be carried out in a predictable order.
To do so, an appropriate monad must be defined,
this approach also addresses the aliasing problem.

The session interaction monad hides the channel from the programmer,
preventing the creation of aliases that could grant non-linear access to the channel.

*This section makes an interesting point when discussing this implementation,
referring to using the compiler to generate the appropriate monadic API.
Read [Toninho et al.](#monadic-integration) and [Deniélou et al.](#multirole)*

## Singularity OS

Briefly, Singulary OS is a prototype of a reliable operating system where isolated processes share the same address space, the *exchange heap*.

In Singularity OS, channels are formed as pairs of related endpoints, the channel peers.
A message is sent over a peer is received from the other peers.
Communication is asynchronous and synchronization is made via handshake protocols.

### Channel Contracts in Sing#

Channel communication is ruled by channel contracts,
these are verified at compile-time and describe messages, argument types and valid interactions as finite STMs.
Channel contracts can be viewed as a syntactic variation of session types.

Static analysis in Sing# aims at providing strong guarantees of the absence of errors deriving from communications and the usage of head-allocated objects.
Regarding communication, code correctness relies on the assumption that the processes using the peer endpoints are able to deal with the message types.
To this end, Sing# provides channel contracts describing the communication patterns that are permitted on a given endpoint.

In Sing#, a contract is made of a finite set of message specifications and a finite set of states linked by transitions.
The state of an endpoint is determined by the state of the associated contract, defining messages to be sent and received.
Communication errors are ruled out by associating the two peers of a channel with complementary types, that is, describing complementary actions.

---

**Notes**

1. A summary of [MOOL](#mool) is available in [[channels-as-objects]].
2. A summary of [Plaid](#plaid) is available is [[typestate-oriented]].

**References**

1. <span id="ml-effects"> Typechecking a multithreaded functional language with session types
    (*Vasco T. Vasconcelos, Simon Gay, António Ravara*) </span>
2. <span id="monadic-integration"> Higher-order processes, functions, and sessions: A monadic integration
    (*Bernardo Toninho, Luís Caires, Frank Pfenning*) </span>
3. <span id="multirole"> Dynamic multirole session types
    (*Pierre-Malo Deniélou, Nobuko Yoshida*) </span>
---


[//begin]: # "Autogenerated link references for markdown compatibility"
[channels-as-objects]: session-types/channels-as-objects "Channels as Objects in Concurrent Object-Oriented Programming"
[typestate-oriented]: typestates/typestate-oriented "Typestate-Oriented Programming"
[//end]: # "Autogenerated link references"