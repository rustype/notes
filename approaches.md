# Approaches

Possible approaches to adding behavioral types to Rust.

- Proc Macros
  - Generate the necessary code from annotations
- DSL
  - Use `macro_rules!` to allow the embedding of a small description language

The approaches described above are not mutually exclusive,
however, while they can be used together, the project should try and achieve simplicity,
refraining from using *black magic* and its partners.

---

## WIP

The most solid approach would be to first defined whether the target abstraction are session types or typestates.
While both have seen implementations in Rust, I now provide a possible approach to take in the case of session types.

### Rusty Session Types — Introductory Approach

First we are required to implement protocols the *hard way* to be able to identify existing problems,
these are focused on the developer side as we are not doing protocol research.

These problems can be (but are not limited to):
- API misuse:
  - Calling `close` on a closed resource.
- Message mismatch
  - Sending duplicate messages, e.g. double `HELLO`.

Session types can address problems as such by encapsulating the interaction with a resource in a session.
This session is linear in the sense that it is consumed as used and cannot go back in time,
in the file example this could be problematic if we are required to reopen the file,
however, recursive session types address this issue.

The hardest challenge regarding session types is ensuring linearity,
however the Rust compiler helps us with lifetimes and its mode semantics:

```rust
impl Session {
    fn send(self, value: T) {
        ...
    }
}
```

In the example below, `send` consumes `self`, rendering it "unusable" afterwards.
If we need to keep using the session we just return it:

```rust
impl Session {
    fn send(self, value: T) -> Session {
        ...
    }
}
```

Before we proceed any further, we need to discuss the current crucial flaw in `Session`, it is not generic.
Sessions should be generic over the encapsulated resource, so, `Session`s, instead of `struct`s, are `trait`s.
Defining behavior over values.

We can implement the file example as follows:

```rust
impl File {
    fn open(self) -> Self { ... }
    fn close(self) -> Self { ... }
}

impl Session for File {
    fn send(self, value: T) -> impl Session {
        match value {
            Open => { self.open() }
            Close => { self.close() }
        }
    }
}
```

Since both `open` and `close` return `File`, which in turn implements `Session`, this should be valid(ish) Rust.
However, this approach fails to provide safety from misuse, a developer can call `close` twice!

To address such problem we do the following:

```rust
trait Session {
    type In;
    type Out;
    fn send(self, value: Self::In) -> Self::Out;
}
```

???

The idea falls flat here.
The problem is related to the fact tha the session type should encompass the whole protocol and not one step of communication.