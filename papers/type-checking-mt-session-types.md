# Type checking a multithreaded functional language with session types

Article written by (*Vasco T. Vasconcelos, Simon J. Gay b, António Ravara*)

---

## Session Types

Notes about Section 2 (Session Types) are written in [[session-types](../notes/session-types.md)].

## Syntax

Most language aspects are illustrated in Section 2,
the remaining are now discussed.

### Channels

An aspect regarding channels which was not discussed is that channels are runtime entities,
created when a `request` meets a `accept`.
To define their operational semantics we need to distinguish between channel ends.

Consider a channel `y`, each end is attributed a polarity `+`/`-`,
a polarized channel is denoted as `yp` where `p` is either `+` or `-`.

Duality on polarities is represented `~p` which exchanges `+` and `-`.

### Threads

Threads comprise stacks of expressions waiting for evaluation,
possibly with occurrences of the `fork` primitive.

They also establish new sessions (creating new channels through `request`/`accept`) by synchronizing on shared names.

### Configurations

Configurations have four forms:

- `<t>` — A single thread.
- `(C1 | C2)` — A parallel collection of threads.
- `(vx: [S])C` — The declaration of a typed name.
- `(vy)C` — The declaration of a channel.

#### Typed Name Declaration

The declaration of typed names shows up during the reduction of `new` expressions,
it can also be used at top-level to give a more realistic model of the example in
[[session-types#sharing-names](../notes/session-types.md#sharing-names)].

The configuration below arises during reduction of the thread `<system`.
Defining it in this way represents a situation in which the clients and server are defined as separate components which already shared the name `n`.

```
systemConf = (vn: [&<add: ..., neg: ..., eval: ...>])
             (<negClient n> | <negClient n> | <server n>)
```

### Type Syntax

- `T` — Term type
- `D` — Data type
- `S` — Session type
- `Z` — Channel environment

The type `Chan a` represents the type of the channel with identity `a`;
the session type associated with `a` is recorded separately in a channel environment `Z`.

During reduction program variables may be replaced by channels,
implying that `Chan c` may become `Chan y+`.

Among data types we have channel-state annotated functional types `Z; T -> T; Z` and
types for names `[S]` capable of establishing sessions of type `S`.

---

For the full language syntax, refer to the [original paper](https://doi.org/10.1016/j.tcs.2006.06.028).