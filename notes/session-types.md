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

## Establishing a Connection

To allow client and server to communicate through a shared channel a client will use `request v` and a server `accept v`.
Both happen in different threads and both threads interact to create a new channel.
The value `v` denotes the common knowledge between two threads, a shared name used to create the new channel.

- `[S]` — denotes the type of the channel name.

We can observe an example below:

```
server' :: [&<add: ..., neg: ..., ...>] -> ()
server' x = let u = accept x in (server u; close u)

negClient' :: [&<add: ..., neg: ..., ...>] -> ()
negClient' x = let u = request x in (negClient u; close u)
```

## Sharing Names

For a name to be known it must first be created and shared.
To create a new potentially shared, name of type `[S]`, we write `new S`.
To distribute the name to a second thread, we `fork` a new thread, in whose code the name occurs.

```
system :: ()
system = let x = new &<add: ..., neg: ..., eval: ...> in
         fork negClient x;
         fork addClient x;
         fork server x
```

## Sending Channels on Channels

Imagine two cooperating clients which interact with the server.
One starts an operation and the other receives the result.

In a concrete fashion,
one client establishes a connection, selecting the `neg` operation and sending the corresponding argument,
the second client should now receive the result.
The first client now needs to provide the second client with the channel to the server.
To do so, both clients share a name type `?(?Int.End).End` and establish a connection with the purpose of transmitting the channel.

```
askNeg :: [< ... >] -> [S] -> ()
askNeg x y =
    let u = request x in
        select neg on u; send 7 on u
        let w = request y in
            send u on w; close w

getNeg :: [S] -> ()
getNeg y =
    let w = accept y in
    let u = receive w in
    let i = receive u in
    close u; close w; { ... }
```

## Channel Aliasing

As soon as the creation and naming of channels becomes separate,
aliasing becomes possible.

```
sendSend u v = send 1 on u; send 2 on v
```

Considering the function above (`sendSend`) a possible use would be:

```
sendTwice :: c: !Int.!Int.End; Chan c -> (); c: End
sendTwice w = sendSend w w
```

## Free Variables in Functions

If we write:

```
sendFree v = send 1 on u; send 2 on v
```

Function `sendSend` becomes `λu.sendFree` we now must consider whether aliasing of `u` and `v` should be allowed.
In the case that it is, we have the following types:

```
sendFree :: c: !Int.!Int.End; Chan c -> (); c: End
sendSend :: c: !Int.!Int.End; Chan c -> Chan c -> (); c: End
```

In the case where aliasing is not allowed, we enforce two different channels `c` and `d`.

```
sendFree :: c: !Int.End, d: !Int.End; Chan c -> (); c: End, d: End
sendSend :: c: !Int.End, d: !Int.End; Chan c -> Chan d -> (); c: End, d: End
```

## Polymorphism

As previously seen, `sendFree` admits a share/not-share kind of polymorphism.
It is also possible to enable channel and session polymorphism.

A channel polymorphism can be seen below, where `S = !Int.!Int.End`,
`sendTwice` must be typed once with channel `c` and another with channel `d`:

```
sendTwiceSendTwice :: c: S, d: S; Chan c → Chan d → (); c: End, d: End
sendTwiceSendTwice x y = sendTwice x; sendTwice y
```

For session polymorphism,
where `sendTwice` must be typed once with `c: !Int.!Int.!Int.!Int.End` and another with `c: !Int.!Int.End`:

```
sendQuad :: c : !Int.!Int.!Int.!Int.End; Chan c → Unit; c : End
sendQuad x = sendTwice x; sendTwice x
```

---

Some symbols may have been adapted from the original paper(s) to conform with Markdown limitations.

Notes done here relate to the following papers:

- Type checking a multithreaded functional language with session types
  (*Vasco T. Vasconcelos, Simon J. Gay, António Ravara*)

---
