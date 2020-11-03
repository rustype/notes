# Behavioural Types: from Theory to Tools

---

**Authors** — Simon Gay, António Ravara

**ISBN** — 978-87-93519-82-4

**File** — [behavioral/RE_9788793519817.pdf](https://github.com/rustype/bibliography/blob/main/behavioral/RE_9788793519817.pdf)

---

Given the book is quite large and not every chapter is important to the matter at hands,
only some chapters are reviewed.

The primary criteria of selection was based on what seemed the most useful,
I looked for type-first approaches and the generation of programs,
these matters are relevant in "Rust-land" to make use of the advanced type-system and macros.

## Type-Based Analysis of Linear Communications

This chapter presents a tool and specification language called Hypha
for the type-based analysis of processes that communicate on linear channels.

The specification language is appropriate for modeling concurrent processes that exchange messages over session channels.
The syntax of the language is present in the original document.

Hypha distinguishes between ephemeral input, written as $u?(x_1, \dots, x_n).P$,
and persistent input, written as $^*u?(x_1, \dots, x_n)$.

In the case of ephemeral input, the process then waits for one message from $u$,
the message is considered to be an $n$-tuple,
and then executes $P$.

For permanent input, which waits for an arbitrary number of messages,
each received message spawns a new copy of $P$.

Hypha also ensures that its programs are deadlock free, however, it is important to note that not every deadlock free program is a valid Hypha program.

A process is deadlock free if every residual of P that cannot reduce further does not contain pending communications.
- **Definition** — $P$ is *deadlock free* if whenever $P~\to^*~\mathtt{new}~c_1,\dots,c_n~\mathtt{in}~Q~\nrightarrow$ we have $\neg \mathsf{wait}(a, Q)$ for every $a$.

A process is lock free if every residual $Q$ of $P$ in which there are pending communications can reduce further to a state in which such operations complete.
- **Definition** — $P$ is *lock free* if whenever $P \to^*~\mathtt{new}~c_1,\dots,c_n~\mathtt{in}~Q$ and $\mathsf{wait}(a,Q)$ there is $R$ such that $Q \to^*R$ and $\mathsf{sync}(a, R)$.

### Type System

As previously referred, Hypha's typesystem ensures that well-type processes are guaranteed to be deadlock free.
To that end, Hypha implements a type reconstruction algorithm for this type system and finds a typing derivation for a given process, provided there is one.

#### Channels

In Hypha, channels have three properties:
- *Polarity* which specifies the operations allowed on the channel, which can be none, input, output, or both input and output.
- The message type, which is an $n$-tuple of values.
- A qualifier $q$ specifying how many time the channel can or must be used according to its polarity.
    - The qualifier can be $*$, meaning the channel can be used any number of times.
    - A qualifier in the form $^h_k$ means that the channel can only be used once.
        In this case, $h$ and $k$ are respectively the *level* and the *tickets* associated with the channel.
        Channels with smaller levels must be used before channels with greater levels,
        a channel with $k$ tickets can be sent as a message on another channel at most $k$ times.

Hypha's typing rules are meant to enforce the following channel properties:

- A service channel with input capability must be used by a replicated input process.
- A linear channel cannot be discarded until both its input and output capabilities have been used.
- An operation on a linear channel cannot block channels with lower or equal level.
- The use of a linear channel cannot be postponed forever.

## Session Types with Linearity in Haskell

This chapter reviews existing literature on Session Types and Haskell,
related on how to exploit the type system to provide static guarantees over programs.

### Introduction

Session types are a kind of behavioral type capturing the communication behavior of concurrent processes.
Commonly, session types capture the sequence of sends and receives performed over a channel and the types of the messages carried by these interactions.

A relevant aspect of session type is their requirement for linear usage of channels,
every send being required to have exactly on receive and vice versa.
This makes the implementation of session types on language that lack linear concepts, a challenge.

### Session Types in Haskell

Haskell provides a library for message-passing concurrency, its primitives are as follows:

```haskell
newChan   :: IO (Chan a)          -- channel creation
readChan  :: Chan a -> IO a       -- read from a channel
writeChan :: Chan a -> a -> IO () -- write to a channel
forkIO    :: IO () -> IO ThreadId -- process forking
```

Functions operate within the IO monad to encapsulate side-effectful computations.
While channel typing ensures *data safety*, communication safety is not ensured.

A significant proportion of communication safety (mainly the order of
interactions) can be enforced with just algebraic data types, polymorphism,
and a type-based encoding of duality.

#### Tracking Send and Receive Actions

```haskell
send  :: Chan (Send a t) -> a -> IO (Chan t)
recv  :: Chan (Recv a t) -> -> IO (a, Chan t)
close :: Chan End -> IO ()

data Send a t = Send a (Chan t)
data Recv a t = Recv a (Chan t)
data End
```

The `send` combinator takes as parameters a channel which can transfer values of type `Send a t` and a value `x` of type `a` returning a new channel which can transfer the values of type `t`.
This is implemented via the constructor `Send`, pairing the value `x` with the new channel `c'`,
sending those on the channel `c`, and returning the new continuation channel `c'`.

The `recv` combinator is somewhat dual to this.
It takes a channel `c` on which is received a pair of a value `x` of type `a` and channel `c'` which can transfer values of type `t`.
The pair `(x, c')` is then returned.

The `close` combinator discards its channel which has only the capability of transferring `End` values, which are uninhabited.

#### Partial Safety via a Type-Level Function for Duality

A possible way to capture duality is via a *type family*.
Type families are primitive recursive functions, with strong syntatic restrictions to enforce termination.
Defining the closed type family `Dual`:

```haskell
type family Dual s where
            Dual (Send a t) = Recv a (Dual t)
            Dual (Recv a t) = Send a (Dual t)
            Dual End        = End
```

Duality is used to type the `fork` operation,
which spawns a process with a fresh channel, returning a channel of the dual type:

```haskell
fork :: Link s => (Chan s -> IO ()) -> IO (Chan (Dual s))
```