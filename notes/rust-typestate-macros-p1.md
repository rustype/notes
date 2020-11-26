# Rusty Typestates - Macros

## Introduction

This article picks up where [[rust-typestate]] left off,
if you haven't read it, it is in your best interest to do so beforehand.

Currently, we have a Drone implementation which respects the possible state transitions at compile-time.
However, such implementation requires the developer to write a lot of code,
and as more states are added, the more and more code is required to be written.

In this article we try to ease that pain using `macro_rules!` to generate code for the developer.
If you are not familiar with Rust's `macro_rules!`, I recommend you take a look at their chapter in [the Rust book](https://doc.rust-lang.org/book/ch19-06-macros.html),
[Rust by example](https://doc.rust-lang.org/rust-by-example/macros.html) and the [Rust reference](https://doc.rust-lang.org/reference/macros.html).
If you want a quick and dirty explanation: *they're like C macros, but better*.

## Finding Ways to Improve

Looking back at our drones and their build process (both the "regular" and "reference-able"),
we notice some patterns.

- The `Drone` struct always takes a `state: PhantomData<State>` field.
- The states are all empty `struct`s.
- We write our drone in an incrementally stricter fashion:
  - We start by allowing any type to implement a concrete `Drone<State>`.
  - Then we constrain possible types to only those that implement the `DroneState` marker trait.
  - Finally, we make it impossible to extend the possible states.

## Breaking New Ground

Before we start, it is important to start simple and build incrementally,
with that in mind lets build upon the simpler `Drone`, without references.

### `Drone` and `PhantomData`

As the good programmers we are, we know that work is evil,
and so we want our macro to build the `struct` for us.
To do so, we need the structure name, generic parameter name.

```rust
macro_rules! typestate_struct {
    ($struct_name:ident, $state_name:ident) => {
        pub struct $struct_name<$state_name> {
            state: std::marker::PhantomData<$state_name>
        }
    }
}
```

Using the macro:
```rust
typestate_struct!(Drone, DroneState);
```

Running `cargo expand` we get:
```rust
pub struct Drone<State> {
    state: std::marker::PhantomData<State>,
}
```

We're still missing the remaining `struct` fields.
To take care of that we can add a body to our macro:

```rust
macro_rules! typestate_struct {
    (($struct_name:ident, $state_name:ident) {$($field:ident:$field_type:ty,)*}) => {
        pub struct $struct_name<$state_name> {
            state: std::marker::PhantomData<$state_name>,
            $($field:$field_type,)*
        }
    }
}
```

Notice that we added `{$($field:ident:$field_type:ty,)*}` to our matching logic and `$($field:$field_type,)*` to our expansion.
We also added extra parentheses to the first two parameters, just to clear the usage.
Usage now looks like:
```rust
typestate_struct!(
    (Drone, State) {
        x: f32,
        y: f32,
    }
);
```

Which expands to:
```rust
pub struct Drone<State> {
    state: std::marker::PhantomData<State>,
    x: f32,
    y: f32,
}
```
<!--
The compiler complains (and rightly so) that `PhantomData` cannot be found in scope.
This is due to our qualified use of `PhantomData` in our macro, as we should not import code for the user.
To fix we just need to add `use std::marker::PhantomData;` to the top of our file. -->

By now we have a simple structure being generated, lets move on to the typestates!

### Generating the Typestates

To generate the typestates we have to consider that not all states are used the same way.
To cope with it, we categorize the states according to strictness:

| Strictness | Description | Implications |
| --- | --- | --- |
| None | Allows users to declare the generic argument for any possible type | All bets are off, state is completely public |
| Constrained | Users are required to make the type implement the respective `State` trait | Avoids "accidental" implementations over the generic state |
| Strict | Users are not able to extend the automata | Avoids the addition of new states to the automata |

For this we will write another macro, we will need to declare the strictness level as well as possible states.
Let us start with the possible states.

```rust
macro_rules! typestates_gen {
   ($($typestate:ident,)+) => {
        $(pub struct $typestate;)+
    }
}
```

Usage looks like:
```rust
typestates_gen!(Idle, Hovering, Flying,);
```

And expands to:
```rust
pub struct Idle;
pub struct Hovering;
pub struct Flying;
```

Looks good! Moving on to the strictness level, we need to support three levels,
we can use the Rust's macros ability to pattern match specific keywords.
Our new macro looks like:

```rust
macro_rules! typestates_gen {
    ([$($typestate:ident,)+]) => {
        $(pub struct $typestate;)+
    };
    (limited [$($typestate:ident,)+]) => {
        typestates_gen!([$($typestate,)+]);
        pub trait Limited {}
        $(impl Limited for $typestate {})+
    };
    (strict [$($typestate:ident,)+]) => {
        typestates_gen!([$($typestate,)+]);
        pub trait Limited : sealed::Sealed {}
        $(impl Limited for $typestate {})+
        mod sealed {
            pub trait Sealed {}
            $(impl Sealed for super::$typestate {})+
        }
    };
}
```

In comparison with the previous macro this is a lot to unpack,
lets break it down branch by branch.

```
([$($typestate:ident,)+]) => {
    $(pub struct $typestate;)+
};
```

The first branch is the same as the previous macro,
we just added square brackets so our keyword can be distinguished from the typestates.

Moving on to the second branch:

```
(limited [$($typestate:ident,)+]) => {
    typestates_gen!([$($typestate,)+]);
    pub trait Limited {}
    $(impl Limited for $typestate {})+
};
```

This branch requires the `limited` keyword and a list of typestates.
First, the macro is performs a recursive call to the first branch to generate the typestate `struct`s,
then the macro generates a `Limited` marker trait and implements it for each generated `struct`.

Finally, we move on to the final branch:

```
(strict [$($typestate:ident,)+]) => {
    typestates_gen!([$($typestate,)+]);
    pub trait Limited : sealed::Sealed {}
    $(impl Limited for $typestate {})+
    mod sealed {
        pub trait Sealed {}
        $(impl Sealed for super::$typestate {})+
    }
};
```

The logic behind this branch is the same as for the second,
recurse into the first branch to generate the empty structures and then add and implement the necessary marker traits.

Usage looks like:
```rust
typestates_gen!(strict[Idle, Hovering, Flying,]);
```

Which expands to:
```rust
pub struct Idle;
pub struct Hovering;
pub struct Flying;
pub trait Limited: sealed::Sealed {}
impl Limited for Idle {}
impl Limited for Hovering {}
impl Limited for Flying {}
mod sealed {
    pub trait Sealed {}
    impl Sealed for super::Idle {}
    impl Sealed for super::Hovering {}
    impl Sealed for super::Flying {}
}
```

#### Problems

Currently, our macros are independent of each other, this implies that while we can implement the types for our `Drone`, like the following snippet:

```rust
impl Drone<Idle> {}
```

We can still write unbounded implementations, such as:

```rust
impl Drone<u8> {}
```

Furthermore, there are several places where the implementation assumes what the user wants,
for example, the `strict` typestates generate a module named `sealed`,
we need to add the ability to customize such details.

---

If you're interested in the continuation of this series, please head on to [[rust-typestate-macros-p2]].

[//begin]: # "Autogenerated link references for markdown compatibility"
[rust-typestate]: rust-typestate.md "Rusty Typestates"
[rust-typestate-macros-p2]: rust-typestate-macros-p2.md "Rusty Typestates - (More) Macros"
[//end]: # "Autogenerated link references"