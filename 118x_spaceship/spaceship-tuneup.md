Title: Spaceship needs a tune-up
Subtitle: Addressing some discovered issues with P0515 and P1185
Document-Number: D1630R0
Authors: Barry Revzin, barry dot revzin at gmail dot com
Audience: CWG, EWG

# Introduction

The introduction of `operator<=>` into the language ([P0515R3](https://wg21.link/p0515R3) with relevant extension [P0905R1](https://wg21.link/p0905r1)) added a novel aspect to name lookup: candidate functions can now include both candidates with different names and a reversed order of arguments. The expression `a < b` used to always only find candidates like `operator<(a, b)` and `a.operator<(b)` now also finds `(a <=> b) < 0` and `0 < (b <=> a)`. This change makes it much easier to write comparisons - since you only need to write the one `operator<=>`.

However, that ended up being insufficient due to the problems pointed out in [P1190](https://wg21.link/p1190R0), and in response [P1185R2](https://wg21.link/p1185R2) was adopted in Kona which made the following changes:

1. Changing candidate sets for equality and inequality  
  a. `<=>` is no longer a candidate for either equality or inequality  
  b. `==` gains `<=>`'s ability for both reversed and rewritten candidates  
2. Defaulted `==` does memberwise equality, defaulted `!=` invokes `==` instead of `<=>`.  
3. Strong structural equality is defined in terms of `==` instead of `<=>`  
4. Defaulted `<=>` can also implicitly declare defaulted `==`

Between P0515 and P1185, several issues have come up in the reflectors that this paper hopes to address. These issues are largely independent from each other, and will be discussed independently. 

# Tomasz's example ([CWG 2407][CWG2407])

Consider the following example (note that the use of `int` is not important, simply that we have two types, one of which is implicitly convertible to the other):

    :::cpp
    struct A {
      operator int() const;
    };

    bool operator==(A, int);              // #1
    // builtin bool operator==(int, int); // #2
    // builtin bool operator!=(int, int); // #3

    int check(A x, A y) {
      return (x == y) +  // In C++17, calls #1; in C++20, ambiguous between #1 and reversed #1
        (10 == x) +      // In C++17, calls #2; in C++20, calls #1
        (10 != x);       // In C++17, calls #3; in C++20, calls #1
    }    

There are two separate issues demonstrated in this example: code that changes which function gets called, and code that becomes ambiguous.

## Changing the result of overload resolution

The expression `10 == x` in C++17 had only one viable candidate: `operator==(int, int)`, converting the `A` to an `int`. But in C++20, due to P1185, equality and inequality get reversed candidates as well. Since equality is symmetric, `10 == x` is an equivalent expression to `x == 10`, and we consider both forms. This gives us two candidates:

    :::cpp
    bool operator==(int, A);   // #1 (reversed)
    bool operator==(int, int); // #2 (builtin)
    
The first is an Exact Match, whereas the second requires a Conversion, so the first is the best viable candidate. 

Silently changing which function gets executed is facially the worst thing we can do, but in this particular situation doesn't seem that bad. We're already in a situation where, in C++17, `x == 10` and `10 == x` invoke different kinds of functions (the former invokes a user-defined function, the latter a builtin) and if those two give different answers, that seems like an inherently questionable program. 

The inequality expression behaves the same way. In C++17, `10 != x` had only one viable candidate: the `operator!=(int, int)` builtin, but in C++20 also acquires the reversed and rewritten candidate `(x == 10) ? false : true`, which would be an Exact Match. Here, the status quo was that `x != 10` and `10 != x` both invoke the same function - but again, if that function gave a different answer from `!(x == 10)` or `!(10 == x)`, that seems suspect. 

## Code that becomes ambiguous

The homogeneous comparison is more interesting. `x == y` in C++17 had only one candidate: `operator==(A, int)`, converting `y` to an `int`. But in C++20, it now has two:

    :::cpp
    bool operator==(A, int); // #1
    bool operator==(int, A); // #1 reversed

The first candidate has an Exact Match in the 1st argument and a Conversion in the 2nd, the second candidate has a Conversion in the 1st argument and an Exact Match in the 2nd. While we do have a tiebreaker to choose the non-reversed candidate over the reversed candidate ([\[over.match.best\]/2.9](http://eel.is/c++draft/over.match.best#2.9)), that only happens when each argument's conversion sequence _is not worse than_ the other candidates' ([\[over.match.best\]/2](http://eel.is/c++draft/over.match.best#2))... and that's just not the case here. We have one better sequence and one worse sequence, each way.

As a result, this becomes ambiguous.

Note that the same thing can happen with `<=>` in a similar situation:

    :::cpp
    struct C {
        operator int() const;
        strong_ordering operator<=>(int) const;
    };
    
    C{} <=> C{}; // error: ambiguous
    
But in this case, it's completely new code which is ambiguous - rather than existing, functional code. 

## Similar examples

There are several other examples in this vein that are important to keep in mind, courtesy of Davis Herring.

    :::cpp
    struct B {
        B(int);
    };
    bool operator==(B,B);
    bool f() {return B()==0;}
    
We want this example to work, regardless of whatever rule changes we pursue. One potential rule change under consideration was reversing the arguments rather than parameters, which would lead to the above becoming ambiguous between the two argument orderings.

Also:

    ::cpp
    struct C {operator int();};
    struct D : C {};

    bool operator==(const C&,int);

    bool g() {return D()==C();}
    
The normal candidate has Conversion and User, the reversed parameter candidate has User and Exact Match, which makes this similar to Tomasz's example: valid in C++17, ambiguous in C++20 under the status quo rules.

## Proposal

The model around comparisons is better in the working draft than it was in C++17. We're also now in a position where it's simply much easier to write comparisons for types - we no longer have to live in this world where everybody only declares `operator<` for their types and then everybody writes algorithms that pretend that only `<` exists. Or, more relevantly, where everybody only declares `operator==` for their types and nobody uses `!=`. This is a Good Thing. 

Coming up with a way to design the rewrite rules in a way that makes satisfies both Tomasz's and Davis's examples leads to a _very_ complex set of rules, all to fix code that is fundamentally ambiguous. 

This paper proposes that the status quo is the very best of the quos. Some code will fail to compile, that code can be easily fixed by adding either a homogeneous comparison operator or, if not that, doing an explicit conversion at the call sites. This lets us have the best language rules for the long future this language still has ahead of it. Instead, we add an Annex C entry.

# Cameron's Example

Cameron daCamara submitted the [following example][cameron.sfinae] after MSVC implemented `operator<=>` and P1185R2:

    :::cpp
    template <typename Lhs, typename Rhs>
    struct BinaryHelper {
      using UnderLhs = typename Lhs::Scalar;
      using UnderRhs = typename Rhs::Scalar;
      operator bool() const;
    };

    struct OnlyEq {
      using Scalar = int;
      template <typename Rhs>
      const BinaryHelper<OnlyEq, Rhs> operator==(const Rhs&) const;
    };
     
    template <typename...>
    using void_t = void;
     
    template <typename T>
    constexpr T& declval();
     
    template <typename, typename = void>

    constexpr bool has_neq_operation = false;

    template <typename T>
    constexpr bool has_neq_operation<T, void_t<decltype(declval<T>() != declval<int>())>> = true;

    static_assert(!has_neq_operation<OnlyEq>);
    
In C++17, this example compiles fine. `OnlyEq` has no `operator!=` candidate at all. But, the wording in [over.match.oper] currently states that:

> ... the rewritten candidates include all member, non-member, and built-in candidates for the operator `==` for which the rewritten expression `(x == y)` is well-formed when contextually converted to `bool` using that operator `==`. 

Checking to see whether `OnlyEq`'s `operator==`'s result is contextually convertible to `bool` is not SFINAE-friendly; it is an error outside of the immediate context of the substitution. As a result, this well-formed C++17 program becomes ill-formed in C++20.

The problem here in particular is that C++20 is linking together the semantics of `==` and `!=` in a way that they were not linked before -- which leads to errors in situations where there was no intent for them to have been linked.

## Proposal

This example we must address, and the best way to address is to carve out less space for rewrite candidates. The current rule is too broad:

> For the `!=` operator ([expr.eq]), the rewritten candidates include all member, non-member, and built-in candidates for the operator `==` for which the rewritten expression `(x == y)` is well-formed when **contextually converted to `bool`** using that operator `==`. For the equality operators, the rewritten candidates also include a synthesized candidate, with the order of the two parameters reversed, for each member, non-member, and built-in candidate for the operator == for which the rewritten expression `(y == x)` is well-formed when **contextually converted to `bool`** using that operator `==`.

We really don't need "contextually converted to `bool`" - in fact, we're probably not even getting any benefit as a language from taking that broad a stance. After all, if you wrote a `operator==` that returned `std::true_type`, you probably have good reasons for that and don't necessarily want an `operator!=` that returns just `bool`. And for types that are even less `bool`-like than `std::true_type`, this consideration makes even less sense.

This paper proposes that we reduce this scope to _just_ those cases where `(x == y)` is a valid expression that has type exactly `bool`. This unbreaks Cameron's example -- `BinaryHelper<OnlyEq, Rhs>` is definitely not `bool` and so `OnlyEq` continues to have no `!=` candidates -- while also both simplifying the language rule, simplifying the specification, and not reducing the usability of the rule at all. Win, win, win.

# Richard's Example

Daveed Vandevoorde pointed out that the wording in [over.match.oper] for determining rewritten candidates is currently:

>  For the relational (7.6.9) operators, the rewritten candidates include all member, non-member, and built-in candidates for the operator `<=>` for which the rewritten expression `(x <=> y) @ 0` is **well-formed** using that `operator<=>`.

Well-formed is poor word choice here, as that implies that we would have to fully instantiate both the `<=>` invocation and the `@` invocation. What we really want to do is simply check if this is viable in a SFINAE-like manner. This led Richard Smith to submit [the following example][smith.unoverloadish]:

    :::cpp
    struct Base { 
      friend bool operator<(const Base&, const Base&); 
      friend bool operator==(const Base&, const Base&); 
    }; 
    struct Derived : Base { 
      friend std::strong_equality operator<=>(const Derived&, const Derived&); 
    }; 
    bool f(Derived d1, Derived d2) { return d1 < d2; } 

The question here is: should we even consider the `@ 0` part of the rewritten expression for validity? The status quo is that we do consider it -- `Derived`'s `operator<=>` is not a viable candidate (because `(d1 <=> d2) < 0` is not a valid expression) and `d1 < d2` invokes `Base`'s `operator<`. If we _do not_ consider it, then `Derived`'s `operator<=>` would be a better match leading to the expression being ill-formed due to the subsequent `< 0`.

The reasoning here is that by considering the `@ 0` part of the expression for determining viable candidates, we are effectively overloading on return type. That doesn't seem right.

## Proposal

This paper agrees with Richard that we should not consider the validity of the `@ 0` part of the comparison in determining the candidate set, and additionally fixes the word choice with "well-formed".

# Default comparisons for reference data members

The last issue, also raised by [Daveed Vandevoorde][vdv.references] is what should happen for the case where we try to default a comparison for a class that has data members of reference type:

    :::cpp
    struct A {
        int const& r;
        auto operator<=>(A const&, A const&) = default;
    };

What should that do? The current wording in [class.compare.default] talks about a list of subobjects, and reference members aren't actually subobjects, so it's not clear what the intent is. There are three behaviors that such a defaulted comparison could have:

1. The comparison could be defined as deleted (following copy assignment with reference data members)
2. The comparison could compare the identity of the referent (following copy construction with reference data members)
3. The comparison could compare through the reference (following what rote expression substitution would do)

In other words:

    :::cpp
    int i = 0, j = 0, k = 1;
                  // |  option 1  | option 2 | option 3 |
    A{i} == A{i}; // | ill-formed |   true   |   true   |
    A{i} == A{j}; // | ill-formed |   false  |   true   |
    A{i} == A{k}; // | ill-formed |   false  |   false  |

Note however that reference data members add one more quirk in conjunction with [P0732](https://wg21.link/p0732r2): does `A` count as having strong structural equality, and what would it mean for:

    :::cpp
    template <int&> struct X { };
    template <A> struct Y { };
    static int i = 0, j = 0;
    X<i> xi;
    X<j> xj;
    
    Y<A{i}> yi;
    Y<A{j}> yj;
    
In even C++17, `xi` and `xj` are both well-formed and have different types. Under option 1 above, the declaration of `Y` is ill-formed because `A` does not have strong structural equality because its `operator==` would be defined as deleted. Under option 2, this would be well-formed and `yi` and `yj` would have different types -- consistent with `xi` and `xj`. Under option 3, `yi` and `yj` would be well-formed but somehow have the same type, which is a bad result. We would need to introduce a special rule that classes with reference data members cannot have strong structural equality. 

## Proposal

This paper proposes making more explicit that defaulted comparisons for classes that have reference data members are defined as deleted. It's the safest rule for now and is most consistent with the design intent. A future standard can always relax this restriction by pursuing option 2 above.

# Wording

Insert a new paragraph after 11.10.1 [class.compare.default]/1:

> <ins>A defaulted comparison operator function for class `C` is defined as deleted if any non-static data member of `C` is of reference type or is an anonymous union.</ins>

Change 11.10.1 [class.compare.default]/3.2:


> A type `C` has _strong structural equality_ if, given a glvalue `x` of type `const C`, either:
> 
- `C` is a non-class type and `x <=> x` is a valid expression of type `std::strong_ordering` or `std::strong_equality`, or
- `C` is a class type with an `==` operator defined as defaulted in the definition of `C`, `x == x` is <del>well-formed when contextually converted to `bool`</del> <ins>a valid expression of type `bool`</ins>, all of `C`'s base class subobjects and non-static data members have strong structural equality, and `C` has no `mutable` or `volatile` subobjects.


Change 12.3.1.2 [over.match.oper]/3.4:

> For the relational ([expr.rel]) operators, the rewritten candidates include all member, non-member, and built-in candidates for the `operator <=>` for which the rewritten expression <del>`(x <=> y) @ 0` is well-formed using that operator`<=>`</del> <ins>`(x <=> y)` is valid</ins>. For the relational ([expr.rel]) and three-way comparison ([expr.spaceship]) operators, the rewritten candidates also include a synthesized candidate, with the order of the two parameters reversed, for each member, non-member, and built-in candidate for the operator `<=>` for which the rewritten expression <del>`0 @ (y <=> x)` is well-formed using that `operator<=>`</del> <ins>`(y <=> x)` is valid</ins>. For the `!=` operator ([expr.eq]), the rewritten candidates include all member, non-member, and built-in candidates for the operator == for which the rewritten expression `(x == y)` is <del>well-formed when contextually converted to `bool` using that operator `==`</del> <ins>valid and of type `bool`</ins>. For the equality operators, the rewritten candidates also include a synthesized candidate, with the order of the two parameters reversed, for each member, non-member, and built-in candidate for the operator `==` for which the rewritten expression `(y == x)` is <del>well-formed when contextually converted to `bool` using that operator `==`</del> <ins>valid and of type `bool`</ins>. [-Note: A candidate synthesized from a member candidate has its implicit object parameter as the second parameter, thus implicit conversions are considered for the first, but not for the second, parameter. —end note] In each case, rewritten candidates are not considered in the context of the rewritten expression. For all other operators, the rewritten candidate set is empty.

Change 12.3.1.2 [over.match.oper]/8 to use `!` instead of `?:`

> If a rewritten candidate is selected by overload resolution for a relational or three-way comparison operator `@`, `x @ y` is interpreted as the rewritten expression: `0 @ (y <=> x)` if the selected candidate is a synthesized candidate with reversed order of parameters, or `(x <=> y) @ 0` otherwise, using the selected rewritten `operator<=>` candidate. If a rewritten candidate is selected by overload resolution for a `!=` operator, `x != y` is interpreted as <del>`(y == x) ? false : true`</del> <ins>`!(y == x)`</ins> if the selected candidate is a synthesized candidate with reversed order of parameters, or <del>`(x == y) ? false : true`</del> <ins>`!(x == y)`</ins> otherwise, using the selected rewritten operator== candidate. If a rewritten candidate is selected by overload resolution for an `==` operator, `x == y` is interpreted as <del>`(y == x) ? true : false`</del> <ins>`(y == x)`</ins> using the selected rewritten `operator==` candidate.

Add a new entry to [diff.cpp17.over]:

<blockquote><p><b>Affected subclause</b>: [over.match.oper]<br />
<b>Change:</b> Equality and inequality expressions can now find reversed and rewritten candidates.<br />
<b>Rationale:</b> Improve consistency of equality with spaceship and make it easier to write the full complement of equality operations.<br />
<b>Effect on original feature:</b> Equality and inequality expressions between two objects of different types, where one is convertible to the other, could change which operator is invoked. Equality and inequality expressions between two objects of the same type could become ambiguous.
<pre><code class="language-cpp">struct A {
  operator int() const;
};

bool operator==(A, int);              // #1
// builtin bool operator==(int, int); // #2
// builtin bool operator!=(int, int); // #3

int check(A x, A y) {
  return (x == y) +  // ill-formed; previously well-formed
    (10 == x) +      // calls #1, previously called #2
    (10 != x);       // calls #1, previously called #3
}</code></pre></blockquote>


    
[CWG2407]: http://wiki.edg.com/pub/Wg21cologne2019/CoreIssuesProcessingTeleconference2019-03-25/cwg_active.html#2407 "CWG 2407: Missing entry in Annex C for defaulted comparison operators||Tomasz Kamiński||Feb 26, 2019"
[cameron.sfinae]: http://lists.isocpp.org/core/2019/04/5935.php "Potential issue after P1185R2 - SFINAE breaking change||Cameron daCamara||April 04, 2019"
[smith.unoverloadish]: http://lists.isocpp.org/core/2019/05/6420.php "Processing relational/spaceship operator rewrites||Richard Smith||May 21, 2019"
[vdv.references]: http://lists.isocpp.org/core/2019/05/6462.php "Generating comparison operators for classes with reference or anonymous union members||Daveed Vandevoorde||May 23, 2019"