---
title: "Designated-initializers for Base Classes"
document: D2287R4
date: today
audience: CWG
author:
    - name: Barry Revzin
      email: <barry.revzin@gmail.com>
toc: true
---

# Revision History

Since [@P2287R3], clarify that the design allows you to initialize the first base class from a non-designated initializer but the second base class's members indirectly. Simplified wording as a result.

[@P2287R2] proposed only allowing indirectly initializing members using the designated syntax. R3 additionally allows directly initializing base class subobjects *without* a designator. Also adds an implementation.

[@P2287R1] proposed a novel way of naming base classes, as well as a way for naming indirect non-static data members. This revision _only_ supports naming direct or indirect non-static data members, with no mechanism to name base classes. Basically only what Matthias suggested.

[@P2287R0] proposed a single syntax for a _designated-initializer_ that identifies a base class. Based on a reflector suggestion from Matthias Stearn, this revision extends the syntax to allow the brace-elision version of _designated-initializer_: allow naming indirect non-static data members as well. Also actually correctly targeting EWG this time.

# Introduction

[@P0017R1] extended aggregates to allow an aggregate to have a base class. [@P0329R4] gave us designated initializers, which allow for much more expressive and functional initialization of aggregates. However, the two do not mix: a designated initializer can currently only refer to a direct non-static data members. This means that if I have a type like:

::: std
```cpp
struct A {
    int a;
};

struct B : A {
    int b;
};
```
:::

While I can initialize an `A` like `A{.a=1}`, I cannot designated-initialize `B`. An attempt like `B{{.a=1}, .b=2}` runs afoul of the rule that the initializers must either be all designated or none designated. But there is currently no way to designate the base class here.

Which means that my only options for initializing a `B` are to fall-back to regular aggregate initialization and write either `B{{1}, 2}` or `B{1, 2}`. Neither are especially satisfactory.

## Design Space

There are basically three potential approaches for being able to designated-initialize `B`:

::: std
```cpp
// provide a way to name the base class
auto b1 = B{.A={.a=1}, .b=2};

// allow mixing designated and non-designated
auto b2 = B{{.a=3}, .b=4};

// allow directly initializing indirect members
auto b3 = B{.a=5, .b=6};
```
:::

A previous revision of this paper proposed allowing only `b1` — coming up with a way to name the base class. This is much more complicated than naming an aggregate member because base classes aren't just `$identifier$`s, they can include template parameters. This revision eschews that approach entirely.

This leaves the other two options. When this paper was discussed in [Varna](https://wiki.edg.com/bin/view/Wg21varna/NotesEWGP2287), there was some desire expressed for having braces around the base class subobject (as in `b2` above). While allowing for optional braces is straightforward enough (and is simply a matter of refining the restriction on mixing designated and non-designated initializers), I would be strongly opposed to mandating braces.

Designated-initialization mirrors assignment. The goal is to do this substitution:

::: cmptable
### Assignment
```cpp
[&]{
    B b;
    b.a = 5;
    b.b = 6;
    return b;
}();
```

### Initialization
```cpp
B{.a=5, .b=6}
```
:::

We can directly assign into `a`, so we should be able to directly initialize into it. Of course, in C the two forms are identical (assuming C had lambdas) while in C++ they aren't necessarily due to the existence of constructors. This makes designated initialization in C a much simpler problem. Plus C doesn't have base classes. But the essence should be the same — mandating braces breaks that.

Additionally, in many cases, the point of inheritance of aggregates is not because we actually need an "is-a" relationship but rather to compose aggregate members. I don't need `B` to be an `A`, I need `B` to have the same members as `A` with the same convenient access syntax — as opposed to having a member of type `A` and needing an extra accessor in the chain. Forcing the user to initialize `B` with braces forces the user to be aware of what is basically an implementation detail. It's frustrating enough to deal with the requirement that initializers be in-order, additionally requiring braces is too much.

Lastly, aggregate initialize _already_ allows for brace elision. `B{1, 2}` is well-formed today, despite the `1` initializing a base class subobject and `2` initializing a direct member. So it strikes me as especially contrary to the design to suddenly mandate braces here.

But _allowing_ (not mandating) braces? That seems totally fine. Also, notably, gcc in `-std=c++17` mode — still to this day on trunk — supports `B{{.a=1}, .b=2}`. We actually had some code break while upgrading to C++20 that initialized aggregates in this way. While I think `B{.a=1, .b=2}` is the clearest way to initialize this type, `B{{.a=1}, .b=2}` is still great — and both are substantial improvements over the best you can do in valid C++20 today, which would be either `B{{.a=1}, 2}` or simply `B{1, 2}`.


# Proposal

This paper proposes extending designated initialization syntax to:

1. allow directly initializing base class aggregate elements, and
2. allow mixing designated and non-designated initializers, if all of the non-designated initializers are at the beginning of the *initializer-list* and are used to initialize base class subobjects.

 In short, based on the above declarations of `A` and `B`, this proposal allows all of the following declarations:

::: std
```cpp
B{{1}, 2}         // already valid in C++17
B{1, 2}           // already valid in C++17

B{.a=1, .b=2}     // proposed
B{{.a=1}, .b=2}   // proposed
B{.a{1}, .b{2}}   // proposed
B{.b=2, .a=1}     // still ill-formed
```
:::

## Naming the base classes

The original revisions of this paper dealt with how to name the `A` base class of `B`, and what this means for more complicated base classes (such at those with template parameters). This revision eschews that approach entirely: it's simpler to just stick with naming members, direct and indirect. After all, that's how these aggregates will be interacted with.

What this means is that while this paper proposes that this works:

::: std
```cpp
struct A { int a; };
struct B : A { int b; };

auto b = B{.a=1, .b=2};
```
:::

And in a type that has a base class that is not an aggregate, you can use the mix-and-match form if you use braces, but not otherwise:

::: std
```cpp
struct C : std::string { int c; };

auto c1 = C{"hello", .c=3};    // ill-formed
auto c2 = C{{"hello"}, .c=3};  // ok
```
:::

If you have a hierarchy with repeated members, you'll likewise have to use the mix-and-match form — but every base class has to be initialized with braces:

::: std
```cpp
struct D { int x; };
struct E : D { int x; };

auto e1 = E{.x=1};         // initializes E::x, not D::x
auto e2 = E{{.x=1}, .x=2}; // initializes D::x to 1 and E::x to 2
auto e3 = E{D{1}, .x=2};   // ill-formed
```
:::

If you have two base classes, they do not need to look the same. It's just that all the non-designated initializers have to be at the front:

::: std
```cpp
struct F { int f; };
struct G { int g; };
struct H : F, G { int h; };

auto h1 = H{{.f=1}, {.g=2}, .h=3}; // ok
auto h2 = H{{.f=1}, .g=2, .h=3};   // ok, not all bases have to be the same
auto h3 = H{{.f=1}, 2, .h=3};      // ill-formed — can only mix designated and non-designated if
                                   // the non-designated ones are braced-init-lists which initialize
                                   // a base class
```
:::

Coming up with a way to _name_ the base class subobject of a class seems useful, but that's largely orthogonal. It can be done later.

## Naming all the subobjects

The current wording we have says that, from [dcl.init.aggr]{.sref}/3.1:

::: std
[3.1]{.pnum} If the initializer list is a brace-enclosed *designated-initializer-list*, the aggregate shall be of class type, the *identifier* in each designator shall name a direct non-static data member of the class [...]
:::

And, from [dcl.init.list]{.sref}/3.1 (conveniently, it's 3.1 in both cases):

::: std
[3.1]{.pnum} If the *braced-init-list* contains a *designated-initializer-list*, `T` shall be an aggregate class. The ordered identifiers in the designators of the *designated-initializer-list* shall form a subsequence of the ordered identifiers in the direct non-static data members of `T`. Aggregate initialization is performed ([dcl.init.aggr]).
:::

The proposal here is to extend both of these rules to cover not just the direct non-static data members of `T` but also all indirect members, such that every interleaving base class is also an aggregate class. That is:

::: std
```cpp
struct A { int a; };
struct B : A { int b; };
struct C : A { C(); int c; };
struct D : C { int d; };

A{.a=1};       // okay since C++17
B{.a=1, .b=2}; // proposed okay, 'a' is a direct member of an aggregate class
               // and A is a direct base
C{.c=1};       // error: C is not an aggregate
D{.a=1};       // error: 'a' is a direct member of an aggregate class
               // but an intermediate base class (C) is not an aggregate
```
:::

Or, put differently, every identifier shall name a non-static data member that is not a (direct or indirect) member of any base class that is not an aggregate.

Also this is still based on lookup rules, so if the same name appears in multiple base classes, then either it's only the most derived one that counts:

::: std
```cpp
struct X { int x; };
struct Y : X { int x; };
Y{.x=1}; // initializes Y::x
```
:::

or is ambiguous:

::: std
```cpp
struct X { int x; };
struct Y { int x; };
struct Z : X, Y { };
Z{.x=1}; // error:: ambiguous which X
```
:::

would be ill-formed on the basis that `Z::x` is ambiguous.

## Impact on Existing Code

There is one case I can think of where code would change meaning:

::: std
```cpp
struct A { int a; };
struct B : A { int b; };

void f(A); // #1
void f(B); // #2

void g() {
    f({.a=1});
}
```
:::

In C++23, `f({.a=1})` calls `#1`, as it's the only viable candidate. But with this change, `#2` also becomes a viable candidate, so this call becomes ambiguous. I have no idea how much such code exists. This is, at least, easy to fix.

I don't think there's a case where code would change from one valid meaning to a different valid meaning - just from valid to ambiguous.

## Implementation Experience

I implemented this [in clang](https://github.com/llvm/llvm-project/compare/main...brevzin:llvm-project:p2287?expand=1) in a very literal way — by synthesizing a new designated-initializer-list to initialize the base classes in the situations where that comes up. There is probably a more straightforward way to do this, but all the examples in the paper work.


# Wording

## Strategy

The wording strategy here is as follows. Let's say we have this simple case:

::: std
```cpp
struct A { int a; };
struct B : A { int b; };
```
:::

In the current wording, `B` has two elements (the `A` direct base class and then the `b` member). The initialization `B{.b=2}` is considered to have one explicitly initialized element (the `b`, initialized with `2`) and then the `A` is not explicitly initialized and cannot have a default member initializer, so it is copy-initialized from `{}`.

The strategy to handle `B{.a=1, .b=2}` is to group the indirect non-static data members under their corresponding direct base class and to treat those base class elements as being explicitly initialized. So here, the `A` element is explicitly initialized from `{.a=1}` and the `b` element continues to be explicitly initialized from `2`. And then this applies recursively, so given:

::: std
```cpp
struct C : B { int c; };
```
:::

With `C{.a=1, .c=2}`, we have:

* the `B` element is explicitly initialized from `{.a=1}`, which leads to:
  * the `A` element is explicitly initialized from `{.a=1}`, which leads to:
    * the `a` element is explicitly initialized from `1`
  * the `b` element is not explicitly initialized and has no default member initializer, so it is copy-initialized from `{}`
* the `c` element is explicitly initialized from `2`



## Actual Wording

Change the grammar in [dcl.init.general]{.sref}/1 to allow a `$designated-initializer-list$` to start with an `$initializer-list$`:

::: std
```diff
+ $braced-init-list-list$:
+   $braced-init-list$
+   $braced-init-list$ , $braced-init-list-list$

+ $designated-only-initializer-list$:
+   $designated-initializer-clause$
+   $designated-only-initializer-list$ , $designated-initializer-clause$

  $designated-initializer-list$:
-   $designated-initializer-clause$
-   $designated-initializer-list$ , $designated-initializer-clause$
+   $designated-only-initializer-list$
+   $braced-init-list-list$ , $designated-only-initializer-list$
```
:::

Add a new term after we define what an aggregate and the elements of an aggregate are:

::: std
[1]{.pnum} An *aggregate* is [...]

[2]{.pnum} The *elements* of an aggregate are: [...]

::: addu
[*]{.pnum} The _designatable members_ of an aggregate `T` are:

* [*.1]{.pnum} For each direct base class `C` of `T` that is itself an aggregate, in the order in which they appear in the `$base-specifier-list$`, the designatable members of `C` for which lookup for that member in `T` finds the member of `C`, followed by
* [*.2]{.pnum} the direct non-static data members of `T` that are not members of an anonymous union, in declaration order.
:::

[3]{.pnum} When an aggregate is initialized [...]
:::

Extend [dcl.init.aggr]{.sref}/3.1:

::: std
[3.1]{.pnum} If the initializer list is a brace-enclosed `$designated-initializer-list$`, the aggregate shall be of class type[, the identifier in each designator shall name a direct non-static data member of the class, and the explicitly initialized elements of the aggregate are the elements that are, or contain, those members.]{.rm} [`C`.]{.addu}

::: addu
The explicitly initialized elements of `$C$` are:

* [3.1.1]{.pnum} For each `$braced-init-list$` in the `$designated-initializer-list$`, the base class subobject of `$C$` which the `$braced-init-list$` appertains to (see below), and
* [3.1.2]{.pnum} for each `$identifier$` in the `$designated-initializer-list$`, the designatable member of `$C$` named by, or containing (in the case of an anonymous union or a base class), that `$identifier$`.

In either case, if there is either no such base class subobject or no such designatable member, the program is ill-formed.
:::

:::

And extend [dcl.init.aggr]{.sref}/4 to cover base class elements (TODO: do we need this special case first bullet? Expand the designated-initializer-list bullet instead of adding a new bullet. Probably only need to change 4.2):

::: std
[4]{.pnum} For each explicitly initialized element:

* [4.1]{.pnum} If the element is an anonymous union member and [...]
* [4.2]{.pnum} Otherwise, if the initializer list is a brace-enclosed `$designated-initializer-list$]`, [then]{.addu}

  * [4.2.1]{.pnum} [the element is initialized with the `$brace-or-equal-initializer$` of the corresponding `$designated-initializer-clause$`]{.rm} [if the corresponding initializer is a `$designated-initializer-clause$`, the element is initialized with that `$brace-or-equal-initializer$`.]{.addu} If that initializer is of the form `= $assignment-expression$` and a narrowing conversion ([dcl.init.list]) is required to convert the expression, the program is ill-formed.
[The form of the initializer determines whether copy-initialization or direct-initialization is performed.]{.note}
  * [4.2.2]{.pnum} [otherwise, the element is initialized with the corresponding `$braced-init-list$`.]{.addu}

  ::: addu
  ::: example
  ```
  struct A { int a; };
  struct B : A { int b; };
  struct C : B { int c; };

  // the A element is initialized from {.a=1}
  B x = B{.a=1};

  // the B element is initialized from {.a=2, .b=3}
  // which leads to its A element being initialized from {.a=2}
  C y = C{.a=2, .b=3, .c=4};

  struct A2 : A { int a; };

  // the A element is not explicitly initialized
  A2 z = {.a=1};

  struct D { int d; };
  struct E { int e; };
  struct F : D, E { int f; };

  // the D element is initialized from {.d=1}
  // the E::e element is initialized from =2
  // the F::f element is initialized from =3
  F f = {{.d=1}, .e=2, .f=3};
  ```
  :::
  :::

* [4.3]{.pnum} Otherwise, the initializer list is a brace-enclosed `$initializer-list$`. [...]

:::

Extend [dcl.init.list]{.sref}/3.1:

::: std
[3.1]{.pnum} If the _braced-init-list_ contains a _designated-initializer-list_, `T` shall be an aggregate class.
The ordered *identifier*s in the designators of the *designated-initializer-list* shall form a subsequence of [the ordered `$identifiers$` in the direct non-static data members]{.rm} [the designatable members [(dcl.init.aggr)] of `T`.]{.addu}

Aggregate initialization is performed ([dcl.init.aggr]).

::: example
```diff
    struct A { int x; int y; int z; };
    A a{.y = 2, .x = 1};                // error: designator order does not match declaration order
    A b{.x = 1, .z = 2};                // OK, b.y initialized to 0

+   struct B : A { int q; };
+   B e{.x = 1, .q = 3};                // OK, e.y and e.z initialized to 0
+   B f{.q = 3, .x = 1};                // error: designator order does not match declaration order

+   struct C { int p; int x; };
+   struct D : A, C { };
+   D g{.y=1, .p=2};                    // OK
+   D h{.x=2};                          // error: x is not a designatable member

+   struct NonAggr { int na; NonAggr(int); };
+   struct E : NonAggr { int e; };
+   E i{.na=1, .e=2};                   // error: na is not a designatable member
```
:::
:::

Add an Annex C entry:

::: std
::: addu
**Affected sublcause**: [dcl.init] <br/>
**Change**: Support for designated initialization of base classes of aggregates. <br/>
**Rationale**: New functionality. <br/>
**Effect on original feature**: Some valid C++23 code may fail to compile. For example:

```
struct A { int a; };
struct B : A { int b; };

void f(A); // #1
void f(B); // #2

void g() {
    f({.a=1}); // OK (calls #1) in C++23, now ill-formed (ambiguous)
}
```
:::
:::

## Feature-test Macro

Bump `__cpp_­designated_­initializers` in [cpp.predefined]{.sref}:

::: std
```diff
- __cpp_­designated_­initializers @[201707L]{.diffdel}@
+ __cpp_­designated_­initializers @[2025XXL]{.diffins}@
```
:::

# Acknowledgements

Thanks to Matthias Stearn for, basically, the proposal. Thanks to Tim Song for helping with design questions and wording.
