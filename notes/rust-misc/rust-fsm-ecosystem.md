---
tags: rust, fsm, typestate
---

# Rust's FSM Ecosystem

This post's purpose is to provide an overview of Rust's existing finite state machine crates and if (and how) they fit in my use case,
it fits the bigger purpose of my ongoing research into [[rust-typestate-index]].

One of the main goals of this project is to be able to provide compile-time errors if the programmer mistakenly calls the wrong transition,
thus ensuring only valid transitions occur.

As a quick summary, typestates' main purpose is to raise some state to the type level,
a classic example is the `File` abstraction, which can be either in the `Open` or `Closed` state.
In the `Open` state, `File`'s API allows the developer to write `.read`, `.write` or `.close`,
while in the `Closed` state it only allows for `.open`,
in this case, typestates prevent attempting to read from a closed file.
Typestates can be encoded as [FSMs](https://en.wikipedia.org/wiki/Finite-state_machine),
hence their relation with the current post.



Crates were discovered searching for `state machine` in [crates.io](https://crates.io).

| Crate Name             | Crates.io                                       | Repository                                       | Documentation                                            | Last Commit                                                                                                   |
| ---------------------- | ----------------------------------------------- | ------------------------------------------------ | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `finite-automata`      | <https://crates.io/crates/finite-automata>      | <https://github.com/RobertDurfee/FiniteAutomata> | <https://docs.rs/finite-automata/0.1.1/finite_automata/> |  |
| `fsm`                  | <https://crates.io/crates/fsm>                  |                                                  |                                                          |                                                                                                               |
| `macro_machine`        | <https://crates.io/crates/macro_machine>        |                                                  |                                                          |                                                                                                               |
| `state_machine_future` | <https://crates.io/crates/state_machine_future> |                                                  |                                                          |                                                                                                               |
| `mode`                 | <https://crates.io/crates/mode>                 |                                                  |                                                          |                                                                                                               |
| `machine`              | <https://crates.io/crates/machine>              |                                                  |                                                          |                                                                                                               |
| `smlang`               | <https://crates.io/crates/smlang>               |                                                  |                                                          |                                                                                                               |
| `rust-fsm-dsl`             | <https://crates.io/crates/rust-fsm-dsl>             |                                                  |                                                          |                                                                                                               |

https://crates.io/crates/microstate
https://crates.io/crates/beehave
https://crates.io/crates/genfsm

## Existing Crates

After the brief introduction to typestates, we can start looking around the ecosystem for FSM crates and check if they implement the notion (or a close one) of typestates.

> **Note** - The reviewed libraries were not thoroughly tested and were reviewed from a "which ideas can I take and improve upon" point.

### `finite-automata`

- [Docs.rs](https://docs.rs/finite-automata/0.1.1/finite_automata/)
- [crates.io](https://crates.io/crates/finite-automata)
- [Repository](https://github.com/RobertDurfee/FiniteAutomata) | [Last Commit - 13 Aug 2020](https://github.com/RobertDurfee/FiniteAutomata/commit/962ef8b6bcbadb4b1a093a6829846225fa70e0cc)

The `finite-automata` crate is a "*collection of extendable finite automata with immutable state and transition data*",
neither the documentation, nor the repository present usage examples.
While functions and structures are documented,
the crates does not appear to provide compile-time errors of state machine misuse.

### `fsm`

- [Docs.rs](https://docs.rs/fsm/0.2.2)
- [crates.io](https://crates.io/crates/fsm)
- [Repository](https://github.com/omaskery/fsm-rs) | [Last Commit - 29 Aug 2015](https://github.com/omaskery/fsm-rs/commit/ca0b1c9e0e07996a9e15d47e0f99c18083aca14d)

The `fsm` crate works by defining states and events as `enums`,
registering them in a state machine and then stepping though the states by feeding events.

The following example was taken from the crate's `README`.

```rust
let mut machine = Machine::new(TurnStyleState::Locked);
machine.add_transition(
	TurnStyleState::Locked, TurnStyleEvent::InsertCoin,
	TurnStyleState::Unlocked, |_,_| println!("unlocked")
);
machine.add_transition(
	TurnStyleState::Unlocked, TurnStyleEvent::Push,
	TurnStyleState::Locked, |_,_| println!("locked")
);
machine.on_event(TurnStyleEvent::InsertCoin);
machine.on_event(TurnStyleEvent::Push);
```

As one can see, since `on_event` only handles events which implement `EnumTag`,
it is possible for the programmer to mistakenly write the wrong event and only notice at runtime.

### `macro_machine`

- [Docs.rs](https://docs.rs/macro_machine/0.2.0/macro_machine/)
- [crates.io](https://crates.io/crates/macro_machine)
- [Repository](https://github.com/VKlayd/rust_fsm_macros) | [Last Commit - 5 Oct 2017](https://github.com/VKlayd/rust_fsm_macros/commit/dd73a0bb235a82d44573ec1d889f08c8f8770784)

This crate provides a single `macro_rules` macro,
which implements a DSL for finite state machine declaration.
Just like `fsm`, `macro_machine` allows for event execution to be done without even limitation on every state,
the `execute` function makes use of an enumeration.

The crate allows the user to write callbacks upon entering and leaving states as well as callbacks for different possible events (called commands in the crate documentation).

### `state-machine-future`

- [Docs.rs](https://docs.rs/state_machine_future/0.2.0/state_machine_future/)
- [crates.io](https://crates.io/crates/state_machine_future)
- [Repository](https://github.com/fitzgen/state_machine_future) |  [Last Commit - 28 Mar 2019](https://github.com/fitzgen/state_machine_future/commit/6bceb8cc2c1178ceafe0745b87a501418ea6600d)

`state_machine_future` provides type-safe futures from state machines along with several other FSM related features, such as:

- Every state is reachable from the start state: there are no useless states.
- There are no states which cannot reach a final state. These states would otherwise lead to infinite loops.
- All state transitions are valid. Attempting to make an invalid state transition fails to type check, thanks to the generated typestates.

Since the crate is aimed at futures, it is required we use it with an async runtime, such as Tokio.
This makes the crate unusable in cases which there is no use for a runtime,
regardless, the crate puts several interesting ideas to practice.

The crate works around a custom derive macro (`#[derive(StateMachineFuture)]`) which is used in conjunction with an `enum` and some attribute macros,
the user defines several states as well as valid transitions between them.
The custom derive will then generate all required boilerplate, the user is only required to implement the transition `impl`s.

### `mode`

- [Docs.rs](https://docs.rs/mode/0.4.0/)
- [crates.io](https://crates.io/crates/mode)
- [Repository](https://github.com/andrewtc/mode) |  [Last Commit - 21 Mar 2020](https://github.com/andrewtc/mode/commit/d8d72e12670b9275ca3e5ab4bd5a9a27c9faaf38)

`mode` is a crate to build finite state machines,
it is simple and short, with less than 200 lines of code and provides attractive features as being written exclusively in safe Rust and using no allocations.
The documentation is very complete, containing examples.
Sadly, the library is not "zero-cost" due to the use of `dyn` traits (<https://doc.rust-lang.org/std/keyword.dyn.html>),
however this is an existing limitation which may be lifted as the Rust compiler evolves (<https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md#limitation-to-freeinherent-functions>).

### `machine`

- [Docs.rs](https://docs.rs/machine/0.3.0)
- [crates.io](https://crates.io/crates/machine)
- [Repository](https://github.com/rust-bakery/machine) | [Last Commit - 14 Jun 2019](https://github.com/rust-bakery/machine/commit/4588a2192d5ca9b60f02c9f710b3ff8058384d6b)

The `machine` crate makes use of function procedural macros to create a DSL for the FSM,
however, it does not provide compile time errors for wrong transitions (such as the others).
From the [provided examples](https://github.com/rust-bakery/machine/blob/master/examples/traffic_light.rs):
The crate provides the generated code as well as a `.dot` file for graph rendering (which is a neat idea that I'm definitely taking with me).

```rust
t = t.on_pass_car(PassCar { count: 7 });
assert_eq!(t, Traffic::error());
```

### `smlang`

- [Docs.rs](https://docs.rs/smlang/0.3.5/smlang)
- [crates.io](https://crates.io/crates/smlang)
- [Respository](https://github.com/korken89/smlang-rs/) | [Last Commit - 28 Oct 2020](https://github.com/korken89/smlang-rs/commit/48c660df3ed0b8098d17e0e52e8c536f60e01b21)

This crate is based on [[Boost::ext].SML](https://boost-ext.github.io/sml/tutorial.html),
at this point most FSM libraries look the same with different DSLs behind them.
In the case of `smlang` the DSL is the same from Boost,
documentation-wise, the crate is lacking, however the repository has examples which can be useful.
Finally, a possibly attractive property of this crate is its `no-std` support.
Just like with previous crates, there is the possibility that the developer calls the wrong transition.

### `rust-fsm-dsl`

- [Docs.rs](https://docs.rs/rust-fsm-dsl/0.4.0/rust_fsm_dsl/)
- [crates.io](https://crates.io/crates/rust-fsm-dsl)
- [Repository](https://github.com/eugene-babichenko/rust-fsm) | [Last Commit - 25 Aug 2020](https://github.com/eugene-babichenko/rust-fsm/commit/ee6dcefeb7cc447858ad92af9824707910a02ed6)

Before I get into details, I have a bone to pick with this crate,
it is separated into two components `rust-fsm` (the library) and `rust-fsm-dsl` (the DSL),
in any case this separation would definitely be welcome,
however one of the first things one notices when reading into the crate's documentation is:

> Initially this library was designed to build an easy to use DSL for defining state machines on top of it.
> Using the DSL will require to connect an additional crate `rust-fsm-dsl` (this is due to limitation of the procedural macros system).

While I'm not experienced enough to bash anyone, I doubt whatever was trying to be achieved couldn't be achieved with a **single** crate,
this problem becomes more severe when using the `rust-fsm-dsl` crate, since this one also requires its counterpart, thus creating a "cycle"...

Just like the others, the crate provides a new DSL and still does not protect the developer from himself.

## Conclusion

From the reviewed crates, the ones that stood out were:

- [`state-machine-future`](#state-machine-future) - for being really complete and taking typestates into account.
- [`mode`](#mode) - for the effort put into avoiding allocations and small size.
- [`machine`](#machine) - for the generation of extra artifacts.

It is clear that typestates are not an explored topic in the Rust community,
which is surprising since the community makes use of typestates in embedded Rust
(see the Embedded Rust Book [Chap. 4.1](https://rust-embedded.github.io/book/static-guarantees/typestate-programming.html)),
and PL topics are usually discussed in the community channels.

[//begin]: # "Autogenerated link references for markdown compatibility"
[rust-typestate-index]: ../rust-typestate-series/rust-typestate-index.md "Rusty Typestates"
[//end]: # "Autogenerated link references"