---
tags: rust, typestate
---
# Rusty Typestates - (More) Macros

> This post is part of a series, you can see an overview of the whole series in [[rust-typestate-index]].

## Recap

In our previous post we left off with two macros,
which almost do what we want, but not quite.
They have several problems:
- No flexibility, they do not allow the user to name the generated traits and modules.
- While close in purpose, they did not intersect,
    that is, expanding both would not yield a relationship between the generated code.
- We have more than one macro. We want some kind of DSL, not several disconnected entry points.

## Building a DSL

I took the most responsible approach and decided to tackle all problems at the same time.
While this approach is clearly not ideal, it helps since we are mixing both macros to build a DSL.

### Defining the DSL

Before we start, we want to shape out our DSL,
we already have an idea with the `limited` and `strict` keywords and the square brackets surrounding the typestates.
Our syntax for the grammar definition is close to the Rust `macro_rules!` one,
which in turn, is related to regex, hence the `$` for the variables and `?`, `+` for quantification.
Without further ado, let's define a pseudo-grammar.

#### Strictness Level

We want our strictness level keywords to be optional, in the case they're missing,
the macro does not expand with constraints.
So we will have something like:

```
( limited | struct ) ?
```

#### Visibility

One of the problems of the previous approach was that it would always generate structures as `pub`,
as not everyone may want that, we need to take care of it.
We can just add a visibility qualifier:

```
$visibility ?
```

#### Structure Name

Moving on to the structure name,
this will be the final `struct` name,
the one the user will actually use.
It is (obviously) mandatory:

```
$struct_name
```

#### States

The interesting part starts here.
We need to define the state name for our structure,
which will act as a generic parameter over the *typestated* structure.

While the state is always required, the user may not care for a custom name,
hence we make it optional and later provide a default name.

Another important note is that both `limited` and `strict` imply the use of a trait bound,
just like the state, the bound ends up being optional since we can provide a default name.

We can use `<>` to delimit these traits,
making our DSL a little Rust-like.
Our definition ends up looking like:

```
$(<$state_name $(:$state_trait_bound)?>)?
```

#### Sealed States

We are not done with states yet!
In the `strict` case we need to ensure that states are not extended by external users.
To do so, the macro will expand to the [sealed trait pattern](https://rust-lang.github.io/api-guidelines/future-proofing.html#sealed-traits-protect-against-downstream-implementations-c-sealed),
for now, the important detail to consider is that we might want to name both the `mod` and `trait`.

Before, lets establish some rules:
- Both `mod` and `trait` names are optional, we can provide a default.
- We can rename the `trait` without renaming the `mod`.
- The `mod` cannot be renamed without renaming the `trait`.

We can use `()` to delimit both `mod` and `trait`, separating them with `::`.
With all this in mind we can define the syntax as follows:

```
($($strict_mod_name::)? $strict_mod_trait)?
```

#### Typestates

At last, we reach the typestates,
they are a bit boring though, simply a non-empty set of identifiers,
delimited by `[]` (inspired by the [Plaid](https://www.cs.cmu.edu/~aldrich/papers/onward2009-state.pdf) programming language).
Their grammar looks like:

```
[$($typestate),+]
```

#### Struct Fields

Finally, we need to add fields to our structure,
without them, the structure would be pretty much useless.
In practice, they function just like the typestates, except they're delimited by `{}` and have an accompanying type.
We can describe them as:

```
{$($struct_field:$struct_field_ty),+}
```

### Writing the Macros

Since we already defined our DSL,
it is time to start writing the implementation!
As previously decided, we will implement it using `macro_rules!`.

> This post assumes you've read the previous one,
> and so it also assumes you're familiar with `macro_rules!`.
> If that is not the case, you can find a series of resources on them in the root post of the series.

While writing the macro I found the approach of working top to bottom to work better for me as it requires less guessing
(of course that in reality I worked top to bottom and bottom to top, learn from my mistakes).

We start slow, defining the macro itself:

```rust
macro_rules! typestate {}
```

Since we have our grammar defined we can pick it up and write our main matching branches:

```rust
macro_rules! typestate {
    (
        $vis:vis
        $struct_name:ident
        <$state_name:ident>
        [$($typestate:ident),+]
        {$($field:ident:$field_ty:ty),*}
    ) => { unimplemented!(); }
    (limited
        $vis:vis
        $struct_name:ident
        <$state_name:ident:$state_trait:ident>
        [$($typestate:ident),+]
        {$($field:ident:$field_ty:ty),*}
    ) => { unimplemented!(); }
    (strict
        $vis:vis
        $struct_name:ident
        <$state_name:ident:$state_trait:ident>
        ($sealed_mod:ident::$sealed_trait:ident)
        [$($typestate:ident),+]
        {$($field:ident:$field_ty:ty),*}
    ) => { unimplemented!(); }
}
```

I know this is quite a bit to unpack,
the good news is that if you understand these,
the others just have fewer parameters,
lets go through them one by one.

#### Unrestricted Branch

The first branch is the simplest one:
- `$vis:vis` allows us to capture a visibility modifier, which we will use for our `struct`.
- `$struct_name:ident` captures the name for the structure.
- `<$state_name:ident>` captures the name for the structure state, this will be our generic parameter (e.g. `Struct<$struct_name>`). In this case the state does not use trait bounds to restrict implementation.
- `[$($typestate:ident),+]` captures the typestates to be generated.
- `{$($field:ident:$field_ty:ty),*}` captures the structure fields and their respective types.

Now that we demystified the macro's parameters we are ready to write the implementation.
We will need a base structure and the respective typestates.

##### The Base Structure

We declare `__state` as `PhantomData` since our generic parameter is not actually used by any field,
we are required to use the qualified name since importing could result in duplicate import statements.
Finally, the underscores are used to avoid name clashes.

```
$vis struct $struct_name<$state_name> {
    __state: std::marker::PhantomData<$state_name>,
    $($field:$field_ty,)*
}
```

##### The Typestates

For the typestates we just want to implement each of them as an empty `struct`,
the macro is a classic repetition example.

```
$($vis struct $typestate;)+
```

##### Final Result

We can just take the previous snippets and implement our first branch as:

```rust
macro_rules! typestate {
    (
        $vis:vis
        $struct_name:ident
        <$state_name:ident>
        [$($typestate:ident),+]
        {$($field:ident:$field_ty:ty),*}
    ) => {
        $vis struct $struct_name<$state_name> {
            __state: std::marker::PhantomData<$state_name>,
            $($field:$field_ty,)*
        }
        $($vis struct $typestate;)+
    };
}
```

#### Limited Branch

For the limited branch we have some new elements:

- `limited` is written as a keyword, forcing any possible match to contain it.
- `<$state_name:ident:$state_trait:ident>` is an extended version of the state parameter from our base case, it now includes the type bound over the state.

Just like the state type parameter, the macro's implementation is just a small extension away.

##### Type Bound

To add the type bound to the base structure we can "just add" a `where`:

```
$vis struct $struct_name<$state_name>
where
    $state_name : $state_trait,
{
    __state: std::marker::PhantomData<$state_name>,
    $($field:$field_ty,)*
}
```

And we are done!

##### Type Bound Implementation

To make our states work with our bound we need to generate the new trait
and it for each state, there is no new technique:

```
$vis trait $trait_name {}
$(impl $trait_name for $typestate {})+
```

The first line will generate our trait (e.g. `pub trait TypeBound {}`),
the second will implement it for each typestate (e.g. `impl TypeBound for TypeState {}`)

##### Putting It All Together

Below we see the final branch:

```rust
macro_rules! typestate {
    (limited
        $vis:vis
        $struct_name:ident
        <$state_name:ident:$state_trait:ident>
        [$($typestate:ident),+]
        {$($field:ident:$field_ty:ty),*}
    ) => {
        $vis struct $struct_name<$state_name>
        where
            $state_name : $state_trait,
        {
            __state: std::marker::PhantomData<$state_name>,
            $($field:$field_ty,)*
        }
        $($vis struct $typestate;)+
        $vis trait $trait_name {}
        $(impl $trait_name for $typestate {})+
    };
}
```

There is nothing special to discuss, however we still need to implement the optional parameters!
To do so, we just create a branch without the optional parameters and recurse into the more complex branch with some default values.

Making the type bound optional is as simple as removing it from the matching statement and declaring a default value when performing the recursive call:

```rust
macro_rules! typestate {
    (limited
        $vis:vis
        $struct_name:ident
        <$state_name:ident>
        [$($typestate:ident),+]
        {$($field:ident:$field_ty:ty),*}
    ) => {
        typestate!(
            $vis
            $struct_name
            <$state_name : Limited> // Limited is our default value for the type bound name
            [$($typestate),+]
            {$($field:$field_ty),*}
        );
    };
}
```

#### Strict Branch

For the strict branch we need to implement the sealed trait pattern,
that implies picking our limited implementation and adding some extra information.
We also have new members in our matching logic:

- `($sealed_mod:ident::$sealed_trait:ident)` which will match the new sealed module and trait, separated by `::`.

Before we move on, lets just review what needs to be changed and what doesn't.
We need to add a new private module,
which will contain a `trait` and its implementations for each typestate;
we need to ensure that the existing `trait` (our type bound, from the `limited`) is bounded by the new private trait.
Apart from this, we do not need to make further changes.

##### The Module

The module is fairly simple,
as usual we use the repeating pattern to handle all typestates and
declare the `mod` and `trait` by adding the meta variables to their declarations.

```
mod $sealed_mod {
    pub trait $sealed_trait {}
    $(impl $sealed_trait for super::$typestate {})+
}
```

##### The Sealed Bound

In this case we take the `limited` trait and extend it with a type bound,
the new sealed trait.

```
$vis trait $state_trait : $sealed_mod::$sealed_trait {}
```

##### Finishing Up

Finally, we put everything together:

```rust
macro_rules! typestate {
    (strict
        $vis:vis
        $struct_name:ident
        <$state_name:ident:$state_trait:ident>
        ($sealed_mod:ident::$sealed_trait:ident)
        [$($typestate:ident),+]
        {$($field:ident:$field_ty:ty),*}
    ) => {
        $vis struct $struct_name<$state_name>
        where
            $state_name : $state_trait,
        {
            __state: std::marker::PhantomData<$state_name>,
            $($field:$field_ty,)*
        }
        $($vis struct $typestate;)+
        $vis trait $state_trait : $sealed_mod::$sealed_trait {}
        mod $sealed_mod {
        pub trait $sealed_trait {}
            $(impl $sealed_trait for super::$typestate {})+
        }
        $(impl $trait_name for $typestate {})+
    };
}
```

The default parameters can be added as previously discussed,
shaving existing parameters off and adding defaults to the recursive calls.

### Testing Our DSL

We are ready to test our DSL,
to do so, we pick up [`cargo expand`](https://github.com/dtolnay/cargo-expand) and write a typestate declaration in our code:

```rust
typestate!(
    strict pub Drone <DroneState : StateSet> (state_mod::StateLimit) [Idle, Hovering, Flying] {
        x: f32,
        y: f32
    }
);
```

And call `cargo expand` to get the following result:

```rust
pub struct Idle;
pub struct Hovering;
pub struct Flying;
mod state_mod {
    pub trait StateLimit {}
    impl StateLimit for super::Idle {}
    impl StateLimit for super::Hovering {}
    impl StateLimit for super::Flying {}
}
pub trait StateSet: state_mod::StateLimit {}
impl StateSet for Idle {}
impl StateSet for Hovering {}
impl StateSet for Flying {}
pub struct Drone<DroneState>
where
    DroneState: StateSet,
{
    __state: std::marker::PhantomData<DroneState>,
    x: f32,
    y: f32,
}
```

It works!

## Conclusions

My work on typestates is not finished, I still have ideas to implement and some crates to study (e.g. [state_machine_future](https://github.com/fitzgen/state_machine_future)).

From the work so far, some conclusions have been:

- Implementing typestates using the Rust type system is clearly possible, it's just a question of how ergonomic can they be.
- Writing complex `macro_rules!` is a tough job, they get unwieldy and unreadable *very* fast. Even when I was implementing them, got lost and had to review the whole macro.
- While complex, `macro_rules!` are great to create DSLs. I hope as the ecosystem develops, better macro tooling emerges.

This macro can be found in the [`typestate` crate](https://crates.io/crates/typestate).

[//begin]: # "Autogenerated link references for markdown compatibility"
[rust-typestate-index]: rust-typestate-index.md "Rusty Typestates"
[//end]: # "Autogenerated link references"