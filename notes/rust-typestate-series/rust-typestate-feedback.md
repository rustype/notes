---
tags: rust, typestate
---
# Rusty Typestates - Received Feedback

This page serves as collection of feedback I've deemed useful,
it also features some comments and ideas on the feedback.

## Reddit Post 27/11/2020

### Automata Collections are Unimplementable

One of the most interesting points about using the type system to implement state machines is that some operations become impossible to implement.
An example was storing several `Drone`s in a `Vec`,
the type system will enforce that the `Vec` fixes a generic type:

```rust
let mut drones = Vec::new();
drone.push(Drone::new());               // drones has type Vec<Drone<Idle>>
drone.push(Drone::new().take_off());    // the new drone is of type Drone<Hovering>
```

#### Using `dyn`

A solution posted by the same commenter was defining the type of `drones` as `Vec<dyn StateSet>`.
However, this solution is not adequate since our abstraction now has a runtime cost.

#### Using several collections

Another user suggested the declaration of several data structures for each state,
the drone would then be moved around data structures.
While this solution is fairly easy to implement,
it requires information to be moved around as soon as a state transition occurs,
if it happens too often performance might be impacted.

The implementation of the [`ref_drone`](https://github.com/rustype/drone/blob/383b7d5b7ba3dfac541c7ab9b15acc44fc2f9c6d/src/ref_drone.rs) might mitigate this,
as the state stays "static", this however, is just speculation.

#### Downcasting

A user working with Rust in embedded systems suggested the use of a `degrade(self)` method which returns a flat instance of the structure.
For example:

```rust
let ts_drone: Drone<Idle> = Drone::new();
let simple_drone: SDrone = ts_drone.degrade();
```

This approach has the problem that when you call `degrade`, all bets are off.
However, it aligns with Rust's approach to `unsafe`, where the programmer asks the compiler to trust him.

#### Sidestepping the Problem by Accepting Limitations

Some users suggested that limitations should be well-defined and their existence explained.
I'd say this is the pragmatic approach, however one should not be too discouraged by limitations,
for they are simply markers for how far we've reached, and not how far we can go.

### Procedural Macros

Another suggestion I've received was to use procedural macros rather than `macro_rules!`.
This idea was already in the works, however the `macro_rules!` approach allows me to sketch a quick PoC.