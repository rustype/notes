---
tags: rust, typestate
---
# Rusty Typestates

This work is part of my thesis on behavioral types,
any kind of feedback is welcome and can be submitted as an issue to any of the repositories in the [`rustype` organization](https://github.com/rustype).
For further discussion, please contact me through [Twitter](https://twitter.com/duartejmg) or [Keybase](https://keybase.io/jmgduarte).

This series aims to be a kind of devlog where I explore typestates (maybe others as well)
and their implementation using the Rust type system.

- [[rust-typestate-part-1]]
- [[rust-typestate-part-2]]
- [[rust-typestate-part-3]]
- [[rust-typestate-feedback]]

## Background Reading

### Related Type Theory

Below you can find a list (loosely) organized by recommended reading order.
If you are familiar with the topics, do not be afraid to skip them.
Not all covered topics are directly related with the presented work.

- [*Type Systems*](http://lucacardelli.name/Papers/TypeSystems.pdf)
- [*Typestate-Oriented Programming*](https://www.cs.cmu.edu/~aldrich/papers/onward2009-state.pdf)
- [*Behavioral Types in Programming Languages*](https://pdfs.semanticscholar.org/4906/0ddb3300a83199bef68260a0ddcbdda60105.pdf)
- [*Session Types for Rust*](https://munksgaard.me/papers/laumann-munksgaard-larsen.pdf)
- [*Implementing Multiparty Session Types in Rust*](http://mrg.doc.ic.ac.uk/publications/implementing-multiparty-session-types-in-rust/main.pdf)


### Rust's Macros

In the case you're not familiar with Rust's `macro_rules!` bellow you can find a series of resources:
- Official Resources
  - **The Rust Book** - <https://doc.rust-lang.org/book/ch19-06-macros.html>
  - **Rust By Example** - <https://doc.rust-lang.org/rust-by-example/macros.html>
  - **The Rust Reference** - <https://doc.rust-lang.org/reference/macros.html> & <https://doc.rust-lang.org/reference/macros-by-example.html>
- Several miscellaneous links (gists, blog posts, etc):
  - **The Little Book of Rust Macros** - <https://danielkeep.github.io/tlborm/book/index.html>
  - **How to Write Hygienic Rust Macros** - <https://gist.github.com/Koxiaet/8c05ebd4e0e9347eb05f265dfb7252e1>

[//begin]: # "Autogenerated link references for markdown compatibility"
[rust-typestate-part-1]: rust-typestate-part-1 "Rusty Typestates - Starting Out"
[rust-typestate-part-2]: rust-typestate-part-2 "Rusty Typestates - Macros"
[rust-typestate-part-3]: rust-typestate-part-3 "Rusty Typestates - (More) Macros"
[rust-typestate-feedback]: rust-typestate-feedback "Rusty Typestates - Received Feedback"
[//end]: # "Autogenerated link references"