# Rusty Typestates

## Introduction

In the present days, as systems become more complex, with more moving parts,
being able to ensure that each part cooperates in unison is extremely important.

One of the key ideas from the last years of PL research revolves around making our type-systems smarter,
moving reasoning from the programmer to the compiler.
Rust's borrow checker is a prime example of this, for those which are not familiar,
in a simple manner, Rust is able to reason about memory usage,
ditching manual memory management and garbage collectors.

## Behavioral Types

In the same research lineage, we find behavioral types,
these are meant to describe the behavior of the program,
rather than just the types of data moving around the program.

Further along the road, there are two main topics,
typestates and session types, for now I focus solely on typestates.

## Typestates

Typestates are the idea that types should describe the state of the program,
they can be described as finite state machines, making them easy to verify,
they also provide "call-safety", that is, when using typestates,
if a method is called, the program is in a state which allows so.

For example, consider Java's [`Scanner`](https://docs.oracle.com/javase/8/docs/api/java/util/Scanner.html),
nothing stops you from writing the following code:

```java
Scanner s = new Scanner(System.in);
s.close();
s.nextLine();
```

Resulting in a `IllegalStateException`,
we should be able to do better, that is,
after calling close there should be no doubt that trying to read from the `Scanner` is an error.
Some IDEs might warn you, but not everyone uses an IDE, neither every language has one.

To do better we can use typestates!
If Java had typestates the above example could look like:

```java
Scanner<Open> sOpen = Scanner(System.in);
// `sOpen.close()` consumes the original `sOpen` and returns a new Scanner
Scanner<Closed> sClosed = sOpen.close();
sClosed.nextLine(); // Type error, Scanner<Closed> does not implement `nextLine`
```

Before we move on, it is important to note that typestates require aliasing control,
that is, if we have aliases to `sOpen` we cannot ensure someone else does not call `nextLine`.
We are required to consume `sOpen` when `close` is called,
while Java is unable to provide this mechanism, Rust is.

## Typestates in Rust

Using Rust we will be implementing something more complex than a `Open`/`Closed` switch,
the following example is taken from the paper ["Typestates to Automata and back: a tool"](https://arxiv.org/pdf/2009.08769.pdf).

We have a `Drone` which can be in three states:
- `Idle` — the drone is on the ground.
- `Hovering` — the drone is stopped mid-air.
- `Flying` — the drone is flying from a place to another.

Possible transitions are:
- From `Idle` to `Hovering`, through the `take_off` method.
- From `Hovering` to `Idle`, through the `land` method.
- From `Hovering` to `Flying`, during `move_to` flight from a place to another.
- From `Flying` to `Hovering`, after the `move_to` method.

We can start by defining our `Drone` and the possible states:

```rust
struct Idle;
struct Hovering;
struct Flying;
struct Drone<State> {
    x: f32,
    y: f32,
    state: PhantomData<State>,
}
```

We now need to implement `Drone<State>` for each of our states:

```rust
impl Drone<Idle> {
    pub fn new() -> Self {
        Self {
            x: 0.0,
            y: 0.0,
            state: PhantomData,
        }
    }

    pub fn take_off(self) -> Drone<Hovering> {
        Drone::<Hovering>::from(self)
    }
}

impl Drone<Hovering> {
    fn land(self) -> Drone<Idle> {
        Drone::<Idle>::new()
    }

    fn move_to(self, x: f32, y: f32) -> Drone<Hovering> {
        let drone = Drone::<Flying>::from(self);
        drone.fly(x, y)
    }
}

impl Drone<Flying> {
    fn fly(mut self, x: f32, y: f32) -> Drone<Hovering> {
        self.x = x;
        self.y = y;
        Drone::<Hovering>::from(self)
    }
}
```

Notice that methods consume the structure by using `self` instead of `&self`,
this enables aliasing control, guaranteeing that instances are not reused.
Another important note is the reason behind `move_to` consuming `self` rather than `&self`,
since the input and output types match we should be able to simply use the reference and return `()`.
Given that the typestate is changed from `Hovering` to `Flying` and back during `move_to`,
`self` must be consumed.

> The `From` implementations were left out since they are simple assignments from the old structure to the new one.

Now we are able to safely fly our drone:

```rust
fn drone_flies() {
    let drone = Drone::<Idle>::new() // Drone<Idle>
        .take_off()                  // Drone<Hovering>
        .move_to(-5.0, -5.0)         // => Drone<Flying> => Drone<Hovering>
        .land();                     // Drone<Idle>
    assert_eq!(drone.x, -5.0);
    assert_eq!(drone.y, -5.0);
}
```

If we try to fly our drone before `take_off` or land our drone twice the compiler will correctly point out that the method is not implemented, as seen in the following example:

```rust
fn drone_does_not_fly_idle() {
    let drone = Drone::<Idle>::new();
    drone.move_to(10.0, 10.0); // comptime error: "move_to" is not a member of type Idle
}
```

```
error[E0599]: no method named `move_to` found for struct `drone::Drone<drone::Idle>` in the current scope
   --> src/drone.rs:128:15
    |
4   | / pub struct Drone<State>
5   | | where
6   | |     State: DroneState,
7   | | {
...   |
10  | |     state: PhantomData<State>,
11  | | }
    | |_- method `move_to` not found for this
...
128 |           drone.move_to(10.0, 10.0);
    |                 ^^^^^^^ method not found in `drone::Drone<drone::Idle>`
```

## Improvements

### Stricter States

There still are some possible improvements to be made, as it stands, any type can implement `Drone<State>`,
to provide more security, to avoid someone implementing `Drone<NotAState>` we can bind `State` with a `trait`:

```rust
impl DroneState {}

struct Drone<State> where State: DroneState { ... }
```

Now, if we try to implement `NotAState` we get a compiler error:

```
error[E0277]: the trait bound `drone::NotAState: drone::DroneState` is not satisfied
...
137 | impl Drone<NotAState> {}
    |      ^^^^^^^^^^^^^^^^^^^^ the trait `drone::DroneState` is not implemented for `drone::NotAState`
```

### Even Stricter States

Stricter states grants us more security, however, we may have even stricter requirements!
Our previous implementation disallows states which do not implement `DroneState`,
but an outsider can still implement the trait for a custom state.

To disallow it, we can use [sealed traits](https://rust-lang.github.io/api-guidelines/future-proofing.html#sealed-traits-protect-against-downstream-implementations-c-sealed).

With sealed traits we create a `Sealed` trait, inside a local private module.
Then we implement it for our states, finalizing by adding a type bound to `DroneState`.
Since clients are not able to access the `Sealed` trait, they are not able to add more `DroneState`s:

```rust
mod sealed {
    pub trait Sealed {}
    impl Sealed for super::Idle {}
    impl Sealed for super::Hovering {}
    impl Sealed for super::Flying {}
}
pub trait DroneState: sealed::Sealed {}
```

## Future Work

The described approach relies heavily on moving data around, instead of borrowing references.
In some cases, moving is not ideal, while we could change our approach to rely on `Box<T>`,
this prevents the approach from being used in systems without a heap, such as embedded systems,
which benefit greatly from approaches like these (though they are not the only ones).