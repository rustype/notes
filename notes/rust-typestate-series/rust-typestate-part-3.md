---
tags: rust, typestate
---
# Rusty Typestates - (More) Macros

## Recap

In our previous post we left off with two macros,
which almost do what we want, but not quite.
They have several problems:
- No flexibility, they do not allow the user to name the generated traits and modules.
- While close in purpose, they did not intersect,
    that is, expanding both would not yield a relationship between the generated code.
- We have more than one macro. We want some kind of DSL, not several disconnected entry points.

## Addressing the Issues in the Room

I took the most responsible approach and decided to tackle all problems at the same time.
While this approach is clearly not ideal, it helps since we are mixing both macros to build a DSL.

### Mixing them in a DSL

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
{$($struct_field:$struct_field_type),+}
```