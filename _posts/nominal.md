---
title: 'Nominal Typing Done Wrong'
date: 2024-04-21
permalink: /posts/nominal/
tags:
  - nominal
  - type system
---

# Nominal Typing Done Wrong

When implementing a type system, a question is constantly asked: under what circumstance should we consider two types equal to each other? Two common styles to approach the problem are structural and nominal typing. In essence, structural typing adds a bunch of congruence rules saying that if all components of the two types are equal, then the two types are equal; meanwhile, nominal typing ensures that types have names, and two type are equal only if they have the same name. As we shall see, they have different merits and are usually implemented in different ways. When coming up with a new language, language designers have the freedom of picking different type equality policies for different parts of the language in question. Though it's possible to mix both styles without breaking the static semantics, it's commonly believed that a consistent style can help avoid confusion. Under such assumption, we can roughly divide languages into two kinds: those embracing a structural type system, and those using a nominal one. Then an interesting observation is that I see more languages in real life adopt nominal typing than those structural. In fact, it's hard to think of many languages with a structural type system: OCaml, Scala, Dhall, Typescript, and perhaps Go's interface and Elm's record (sorta). (Un)surprisingly, almost all imperative language's type systems are nominal, with Rust as a representitive. My conjecture for the reason behind the phenomena is that nominal type systems are easier to implement and comprehend.

## Nice Things about Nominal Typing

Nominal typing is all about giving names to types. Or binding free type variables. Or in Conor McBride's words, [natural numbers](https://dl.acm.org/doi/10.1145/1017472.1017477). The best part of nominal typing is that when giving names to new user-defined types, the user are manipulating and extending the equation theory of the host language. As a result, the benefit of giving names boils down to type safety, laziness in the type system, and recursive reasoning.

Let's start with type safety, or rather, "the Newtype Design Pattern". It works by giving the same memory representation a uniquely named type so that terms with the same content must be treated differently in the language. For instance, it enables distinguishing plain numbers with units. In general, it enforces certain type-encoded invariants to be held throughout the program, and provides a better interface for the programmers. The limitation it brings is reasonably the introduction of "boxing/unboxing" of data constructors. Instead of performance impact, the burden is shouldered by the programmers, deliberately - consider implicit type casts as a way of undoing newtypes, but also notice how language like C++ does it terriably wrong.

Two extra merits can be found under the type safety idea of a user-extend type system. For starters, it's easy to introduce controllable subtyping. In contrast to the structural subtyping provided by records, defining a type family of subtyping relation on newtypes that satisfies reflexivity and transitivity can be greatly customized and flexible. Moreover, trait-like features are easier to implement with the presence of newtypes in the type system. The same trait can be given different meanings to different type names, even when they share the same essence.

The next are laziness and recursive reasoning in the type system, which are actually [the same thing](https://dl.acm.org/doi/10.1145/3607853). Nominal typing is suitable for delayed interpretation especially because names are "abstract" in that they can be used before an actual definition is given. Take the design of recursive type for example. A nominal type system tends to use iso-recursive typing, since the named type should not be equal to any other type according to its nominal requirement, which includes being not equal to its own unfolding. The type equations are therefore being "lazy". Only a pair of type abstraction and an application upon should be kicked into a further reduction (according to [NbE](https://www.cse.chalmers.se/~abela/habil.pdf); and a evaluation context rule; the rest of the terms are neutral). Such lazy implementation naturally supports mutual recursion in types; type constructors are also easily implemented since they are neutral, or "uninterpreted", which leads to a simpler normal form.

As a result of the simpler implementation, the laziness brought by nominal typing is also programmer-friendly. It provides the ability to name types before giving definitions, which is natural and identical to the thinking flow of the programmer. The mutual recursion we get for free also liberates the programmer from manual bootstrapping type fix-points. Software-engineering-wise, giving names to types also brings an opportunity to document the code with self-explanatory types.

Nominal type systems are easier to implement and comprehend, as shown above. But is it actually?

## Referential Transparency: Achilles' Heel

We've mentioned Rust as a representitive of a language with a nonimal type system. Now let's see how things can go wrong in `type` alias, a common structural typing feature.

We can start with a small example, a feature flag in Rust: `type_alias_impl_trait`. This feature allows implementing traits for a type alias, treating the new name differently from the aliased type on the right hand side. As shown in the last section, traits in Rust are built around named types. Therefore, types created by `type` alias will turn out to be nominal if the feature flag is on! 

The key lies in referential transparency. The `type` alias feature belongs to the structural side because the new named type should be able to substitude all of its occurence. In this sense, the type aliases are eager and similar to macros. However, type aliases break the promise of "different names are differently typed" made by a pure nominal type system. There didn't exist referential transparency in a simple nominal type system, so adding it is adding eagerness into a lazy system. The trouble emerges when we try to naïvely blend the two paradigms together.

The feature is a disturbing sign that programmers may not be able to fully understand how nominal typing, or how "names" should work, especially when the some names are "unique" while others are not. However, I believe there's a solution that not only makes the type system detail explicitly but moderately exposed, but also support both type aliases and the nice norminal typing interface.

## Sealing Types for More Laziness

The key to resolving the tension between type aliases and nominal typing is to introduce an explicit mechanism for controlling when a type name should be treated as a unique, "sealed" entity versus when it is simply an alias for another type. We can call this mechanism "type sealing".

Starting from a structural type system, we only provide the feature of type definition (alias), but extend the type language by a `seal` type operator. The idea is that when a type is defined, it is initially "unsealed", meaning it is treated as a transparent alias for its underlying type. In this state, the type alias is purely a convenient shorthand and doesn't introduce any new type identity. Referential transparency is maintained. However, the programmer can then explicitly "seal" a type, at which point it takes on its own unique identity and becomes a nominal type distinct from its underlying representation. Once sealed, the type is no longer transparently substitutable and must be treated as its own entity.

For example,

```Haskell
let ints = int * seal int -- ints = int * $1 where $1 <- int
let (a, b): ints = (1, 42)
let (c, d): ints = (2, 7)
let _: ints = (c, b) -- type checks: int * $1 = int * $1
let _: ints = (a, c) -- doesn't: int * int != int * $1
let _: ints = (b, d) -- doesn't: $1 * $1 != int * $1
```

where `=` means equation from substitutions, and `<-` means "assignment" or "allocation". `seal` abstracts the operand away by giving it a unique name (could be a natural number) and allocating a location globally to store it. The neat part is that after such transformation, the name can be substituted and spread around in a referential transparent way, just like anything else in the ambient structural type system, since it's guaranteed to be unique.

This separation of type definition (alias) and sealing allows the programmer to choose, on a per-type basis, whether a type alias should be treated nominally or structurally. It provides fine-grained control over the type system's behavior. The tedium of explicitness can be relieved by introducing a new `let` such as `def`, a syntactic sugar that makes the compiler generate a `let` with `seal` for the user.

The `seal` operator not only provides a way to create nominal types, but also enables automatic boxing based on type annotations. When a value is annotated with a sealed type, the compiler can automatically insert the necessary conversions from (but not to!) the underlying representation. However, if it's bidirectional, the sealing will be kinda useless, since implicit casts to and from the underlying representation will make the sealed type equal to any type that equals to the original type. To solve the problem, we can add a manual `unbox` operator, or just do the normal trick of unboxing upon pattern `match`ing. Overall, it allows the programmer to use sealed types transparently in many contexts, while still benefiting from their distinct identity and type-checking guarantees.

Another advantage of the `seal` operator is that it allows handling recursive types in an equi-recursive way. The sealing brings the right amount of laziness which was brought by the iso-recursive named types. The only difference is that the boundary of laziness is pushed to the `seal` operators.

## Do Nominal Typing via Structural Typing

Nominal typing is a nice interface for the programmer, but not as an implementation strategy. That's why I believe implementing a type system in a nominal way is doing it wrong. But that doesn't mean we can't touch the nice features; all we need is a way to introduce laziness in one way or another. The `seal` type operator is an attempt to let the programmer opt-in to nominal typing for specific type aliases, while retaining the simplicity and referential transparency of structural type definitions by default. In conclusion, it's a solution built upon structural typing for simpler reasoning but ends up possessing the flexibility to borrow power from both nominal and structural way of coding.

