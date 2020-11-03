# Type Systems

---

**Authors** — Luca Cardelli

**DOI** — [N/A](https://doi.org/N/A)

**File** — [type-systems/TypeSystems.pdf](https://github.com/rustype/bibliography/blob/main/type-systems/TypeSystems.pdf)

---

## Introduction

The fundamental purpose of a type system during the running of a program.
When such property holds for all of the program runs that can be expressed within a programming language,
we say that the language is type sound.
A type unsound language will allow a program to crash even though it is judged acceptable by a type checker.

### Language Types

An upper bound on the range of value a variable can take is called a type.

Languages where variables can be given types are called typed languages.
These can be further divided in explicitly typed (if types are part of the syntax) and implicitly typed (if types are optional to the syntax).

Languages that do not restrict the range of variables are called untyped languages:
they do not have types or, equivalently, have a single universal type that contains all values.
In these languages, operations may be applied to inappropriate arguments.

### Execution Errors

#### Safety

It is useful to distinguish between two kinds of errors: errors that stop the computation immediately (trapped errors),
and the ones that go unnoticed (untrapped errors), later causing arbitrary behavior.

A program is considered to be safe if it does not cause untrapped errors to occur.
Languages where all program fragments are safe are called safe languages.

Untyped languages may enforce safety by performing run time checks.
Typed languages may enforce safety by statically rejecting all programs that are potentially unsafe,
they may also use a mixture of runtime and static checks.

Some statically checked languages do not ensure safety.
That is, their set of forbidden errors does not include all untrapped errors.


#### Well-behaved Programs

For any given language, we designate a subset of possible execution errors as forbidden errors,
these should include all untrapped errors and some trapped errors.
A program fragment is said to be well-behaved if it does not cause any forbidden errors.

A language where all programs have good behavior is called strongly checked.
Thus, for such kind of language the following should hold:
- No untrapped errors occur.
- None of the trapped errors designated as forbidden errors occur.
- Other trapped errors may occur, it is the programmer's responsibility to avoid them.

Typed languages can enforce good behavior by means of static checking, preventing unsafe and ill-behaved programs from running.
These languages are statically checked, the checking process is called typechecking and the performing algorithm is the typechecker.
A program which passes the typechecker is said to be well-typed, otherwise it is ill-typed.

Untyped languages can enforce good behavior by performing sufficiently detailed runtime checks.
This process is called dynamic checking,
and these languages are considered strongly checked even though they have neither static checking, nor a type system.

### Type System Properties

Language types range from informal comments to formal specifications subject to theorem proving.
Regular types sit in the middle of the spectrum, being more precise than comments and more easily mechanizable than formal specifications.
Basic properties expected from a type system are:
- Type systems should be decidably verifiable.
    The purpose of a type system is not simply to state programmer intentions, but to actively capture execution errors before they happen.
- Type systems should be transparent: a programmer should be able to easily predict whether a program will typecheck.
- Type systems should be enforceable: type declarations should be statically checked as much as possible, and otherwise dynamically checked.

### How type systems are formalized

Formal type systems are mathematical characterizations of the informal type systems that are described in programming language manuals.
Once a type system is formalized, we can attempt to prove a type soundness theorem stating that well typed programs are well-behaved.

The first step in formalizing a programming language is to describe its syntax.
For most languages of interest this reduces to describing the syntax of types and terms.
Types express static knowledge about programs, whereas terms express algorithmic behavior.

The next step is to define the scoping rules of the language,
which unambiguously associate occurrences of identifiers to their binding locations.
Scoping can be formally specified by defining the set of free variables of a program fragment.
The associated notion of substitution of types or terms for free variables can then be defined.

When this much is settled, one can proceed to define the type of rules of the language.
These describe a relation has-type of the form $M:A$ between terms $M$ and types $A$.
Some languages also require a relation of subtype-of of the form $A<:B$ between types,
and often a relation of equal-type of the form $A=B$ of type equivalence.

The type rules cannot be formalized without first introducing another fundamental ingredient that is not reflected in the syntax of the language: static typing environments.
These are used to record the types of free variables during the processing of program fragments; they correspond closely to the symbol table of a compiler during the typechecking phase.

The final step in formalizing a language is to define its semantics as a relation has-value between terms and a collection of results.

### Type Equality

Considering the example:

```ml
type X = Bool
type Y = Bool
```

If the type names `X` and `Y` match by virtue of being associated with similar types, we have structural equivalence.
If they fail to match by virtue of being distinct type names, we have by-name equivalence.

## The Language of Type Systems

### Judgements

The description of a type system starts with the description of a collection of formal utterances called judgements.
A typical judgement has the form:

$$
\Gamma \vdash \Im
$$

Where $\Im$ is an assertion and its free variables are declared in $\Gamma$.
We say that $\Gamma$ entails $\Im$. Here $\Gamma$ is a static typing environment;
e.g. an ordered list of distinct variables and their types, of the form $\phi, x_1: A_1, \dots, x_n:A_n$.

The empty environment is denoted by $\phi$,
and the collection of variables $x_1 \dots x_n$ declared in $\Gamma$ is indicated by $dom(\Gamma)$, the domain of $\Gamma$.

The form of the assertion varies from judgement to judgement, but all the free variables must be declared in the environment.

The typing judgement asserts that term $M$ has type $A$ with respect to static typing environment for the free variables of $M$.
It has the form:

$$
\Gamma \vdash M:A
$$

Any given judgement can be regarded as *valid* (e.g. $\Gamma \vdash true:Bool$)
or *invalid* (e.g. $\Gamma \vdash true:Nat$).
Validity formalizes the notion of well typed programs.

### Type Rules

Type rules assert the validity of certain judgements on the basis of other judgements that are already known to be valid.
The process gets off the ground by some intrinsically valid judgement.

$$
\frac{\Gamma_1 \vdash \Im_1 \dots \Gamma_n \vdash \Im_n}{\Gamma \vdash \Im}
$$

Each type rule is written as a number of *premise* judgements $\Gamma_i \vdash \Im_i$ above a horizontal line,
with a single *conclusion* judgement $\Gamma \vdash \Im$ bellow the line.
When all of the premises are satisfied, the conclusion must hold; the number of premises may be zero.
Each rule has a name, which by convention, the first word is determined by the conclusion judgement.

### Type Derivations

A derivation in a given type system is a tree of judgements with leaves at the top and a root at the bottom,
where each judgement is obtained from the ones immediately above it by some rule of the system.
A fundamental requirement of type systems is that it must be possible to check whether or not a derivation is properly constructed.

A valid judgement is one that can be obtained as the root of a derivation in a given type system.
That is, a valid judgement is one that can be obtained by correctly applying the type rules.

### Well Typing and Type Inference

In a given type system, a term $M$ is well typed if for an environment $\Gamma$, if there is a type $A$ such that $\Gamma \vdash M : A$ is a valid judgement; that is, if the term M can be given some type.
The discovery of a derivation (and hence of a type) for a term is called the type inference problem.

---

**Notes**

**References**

---