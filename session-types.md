# Session Types

## Input, Output & Sequencing

- `?T` represents *receive* a message of type `T`.
- `!T` represents *send* a message of type `T`.
- `.` represents the sequencing of messages during the protocol.
- `End` indicates the end of the session.

The session type from the server's point of view is:

```
S = ?Int.?Int.!Int.End
```

Considering that the client communicates with the server through a channel `u`,
in ML-style the session may look like:

```
server u =
    let x = receive u in
    let y = receive u in
    send x + y on u
```

We can write the session type for the client by interchanging `?` and `!`.

```
C = !Int.!Int.?Int.End
```

A concrete implementation may look like:

```
client u =
    send 2 on u
    send 3 on u
    let x = receive u in { ... }
```

`{ ... }` is now able to use `x`, while it can use `u`,
the only possible operation at this point is closing the channel.

## Branching Types

Let us extend the previous protocol to support both `add` and `neg`.

- `add` — the client sends two integers and the server replies with their sum.
- `neg` — the client sends a single integer and the server will reply with its negation.

The client must now be able to choose between the available operations.
To this end, we use:

- `&` — the `branch` operator, which indicates that a choice is offered.
- `+*` — the `choice` operator, which indicates that a choice is made.

The server session type now looks like:

```
S = &< add: ?Int.?Int.!Int.End
     , neg: ?Int.!Int.End >
```

To implement the server we introduce the `case` construct:

```
server u =
    case u of {
        add => send (receive u) + (receive u) on u
        neg => send -(receive u) on u
    }
```

Similarly, in the case of the client the session type looks like:

```
C = +*< add: !Int.!Int.?Int.End
      , neg: !Int.?Int.End >
```

To implement the client, the choice is implemented by two functions,
each referring to the choice made by the client.

```
addClient u =
    select add on u
    send 2 on u
    send 3 on u
    let x = receive u in { ... }
```

```
negClient u =
    select neg on u
    send 7 on u
    let x = receive u in { ... }
```

## Error Handling

We now further extend the protocol with a `sqrt` operation,
which receives a `Real` type and returns its square root (a `Real` as well).
This operation can now fail, since the `Real` can be negative.

To model the `sqrt` operation we make use of the `choice` (`+*`) operator,
the new server type is now:

```
S = &< add: ?Int.?Int.!Int.End
     , neg: ?Int.!Int.End
     , sqrt: ?Real.+*<ok: !Real.End, error: End>
     >
```

As well as the client:

```
C = +*< add: !Int.!Int.?Int.End
      , neg: !Int.?Int.End
      , sqrt: !Real.&<ok: ?Real.End, error: End>
      >
```

## Recursive Types

To build a more realistic server we need to allow a session to be a sequence of commands and responses.
To do so, the type must be defined recursively, we also include a `quit` command to allow the session to end.

The new session type now replaces the `End` with `S`.

```
S = &< add: ?Int.?Int.!Int.S
     , neg: ?Int.!Int.S
     , sqrt: ?Real.+*<ok: !Real.S, error: S>
     , quit: End
     >
```

## Function Types

The type of the `server` itself can be expressed as `Chan c -> ()`,
that is a function which consumes a channel client.
To control the usage of the channel type,
function types are annotated with the initial and final type of all used channels.

For functions which do not use a channel the correct notation would be `(_: empty; T -> T'; _: empty)`,
however, for brevity the function signature is abbreviated to `(T -> T')`.

In our ML-like language we can describe the server as follows:

```
server :: (c: &< ... >; Chan c -> (); c : End)
server u =
    case u of {
        ...
    }
```

We can even send function types,
using the function type in the session type signature and the required function parameters.

An example would be a `eval` function:

Whose server-side session type would look like:

```
eval: ?(Int -> Bool).?Int.!Bool.End
```

And the corresponding server implementation code would look like:

```
eval => send (receive u)(receive u) on u
```

As for the client a sample implementation would be:

```
primeClient :: (c: +*<eval: ...>; Chan c -> (); c: End)
primeClient u =
    select eval on u
    send isPrime on u
    send bigNumber on u
    let x = receive u in { ... }
```

---

Some symbols may have been adapted from the original paper(s) to conform with Markdown limitations.

Notes done here relate to the following papers:

- Type checking a multithreaded functional language with session types
  (*Vasco T. Vasconcelos, Simon J. Gay b, António Ravara*)

---
