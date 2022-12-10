<pre class='metadata'>
Title: A type for functions that do not return
Shortname: D####
Revision: 0
!Draft Revision: 0
Audience: LEWG, EWG
Status: D
Group: WG21
URL:
!Source: <a href="https://github.com/ecatmur/no-return/blob/main/paper.md">github.com/ecatmur/no-return/blob/main/paper.md</a>
!Current: <a href="https://htmlpreview.github.io/?https://github.com/ecatmur/no-return/blob/r0/D####R0.html">github.com/ecatmur/no-return/blob/r0/D####R0.html</a>
Editor: Ed Catmur, ed@catmur.uk
Markup Shorthands: markdown yes, biblio yes, markup yes
Abstract:
  Currently, C++ designates functions that do not return to their caller via the <code>[[noreturn]]</code> attribute. This is not visible to the type system
  and so is not amenable to metaprogramming. We propose adding a fundamental type that can be used to signify functions that do not return.
Date: 2023-12-07
</pre>
<pre class='biblio'>
{
    "p0627": {
        "title": "Function to mark unreachable code",
        "href": "https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p0627r6.pdf"
    },
    "wiki": {
        "title": "Bottom type - Wikipedia",
        "href": "https://en.wikipedia.org/wiki/Bottom_type"
    }
}
</pre>

## 1. Changelog

: v0
:: Initial submission

## 2. Introduction

We propose to add a fundamental type, `std::noreturn`, that can be used as the return type of functions that do not return to their caller.
In type theory this is variously called the "bottom" type[[wiki]] or the "empty" type (since it has no values; not to be confused with[[#std-monostate]] empty class types).

## 3. Motivation and Scope

### 3.1 Variant visit

`std::visit` is constrained by requiring that the function object return the same type and value category for all combinations of alternatives.
This is fine, since if we want smarter behavior we can calculate an appropraite return type, for example `std::common_type`:

```c++
template<class F, class... T>
auto common_visit(F visitor, std::variant<T...> const& variant) {
    return std::visit<std::common_type_t<std::invoke_result_t<F, T>...>>(visitor, variant);
    //                ^^^^^^^^^^^^^^^^^^ or std::common_reference_t
    //                ^^^^^^^^^^^^^^^^^^ or std::variant
}
```

However, what if some of the alternatives are inapplicable to a visitor?
    
```c++
std::variant<short, int, double> v;
int i = common_visit([]<class T>(T x) {  // ill-formed: std::common_type<int, int, void> is incomplete
    if constexpr (std::integral<T>)
        return x + 1;
    else
        throw std::runtime_error("expected integral T");
}, v);
```

The visitor returns `void` in such cases, but `std::common_type<int, int, void>` is incomplete.
If we were to assume that `void` indicates nontermination and filter it out, we would have an error when `std::visit` attempts to convert a prvalue of type `void` to `int`.
We might decide to insert `std::unreachable()`[[p0627]] in such cases, but if an intended statement had been omitted by accident we would be promoting a defect from detectable compile time error to undefined runtime behavior.

It would be great if we could tell the type system that the visitor is not expected to return for some alternatives.

### 3.2 Ternary conditional

An interesting aspect of `throw` being an expression is that it is allowed as one of the branches of a ternary conditional:

```c++
auto const x = precondition ? f() : throw std::runtime_error("failed precondition");
```

Since `decltype(throw)` is currently `void`, this is achieved by special casing throw-expression as subexpressions of a ternary conditional expressions; this is somewhat inelegant.

If we decide to terminate the program instead we have a problem, since `std::terminate` has return type `void` and is not a throw-expression:

```c++
auto const x = precondition ? f() : std::terminate();  // ill-formed
```

Workarounds involving comma-expressions or IIFEs are ugly and unclear.
The compiler knows that `std::terminate` doesn't return; why can't we tell the type system as much?

## 3.3 Other languages

Per Wikipedia[[wiki]], several languages with wide or similar usage to C++ employ a "bottom" type: `typing.NoReturn` in Python, `!` in Rust, `noreturn` in D.
The latter is particularly relevant, since D has both `noreturn` and `void`, demonstrating that the combination is meaningful.

## 3.4 Existing implementations

My employer uses a class-typed `NoReturn` type to considerable success; the deficiences of this approach compared to a fundamental type[[#fundamental-vs-class]] inspired this paper.

## 4. Impact On the Standard

### 4.1. Core language

We propose adding a fundamental type, `std::noreturn`. Like `void`, this is a type with no values.

Functions with return type `std::noreturn` do not return to their caller. This means that either they run forever (a non-terminating computation), or they terminate the program, or they exit by throwing an exception.

Expressions of type `cv std::noreturn` can be (implicitly?) converted to any type that can be the type of an expression, including `void`. This is a key difference to `void`, where any expression can be converted *to* `void`; the only valid conversions to `cv std::noreturn` are *from* `cv std::noreturn`.

`decltype(<throw-expression>)` shall change from `void` to `std::noreturn`.

For forming compound types, the same restrictions apply as with `void`: no references, arrays, etc.
In addition, forming pointers to `std::noreturn` is invalid.
The only compound types that can be formed from `std::noreturn` are cv-qualified variants, and function types having `std::noreturn` as their return type.

If implementable, a function or member function with return type `std::noreturn` should be implicitly `[[noreturn]]`.

In a conditional expression, if either branch is of type `std::noreturn`, the result is the type and category of the other branch.
This replaces the existing language on throw expressions. If both branches are of type `std::noreturn`, the result is `std::noreturn`.

A program that creates a value of type `std::noreturn` in evaluated context (including constant evaluation) is ill-formed; there can be no variables, data members, function parameters, non-type template parameters, etc. of type `std::noreturn`.

Like `void`, it it is permitted to return values of type `std::noreturn` from function templates, but otherwise values of type `std::noreturn` cannot be created or stored (whether in a variable or data member).

There are no changes proposed to return type deduction; a function may use `return throw [expression];` to deduce to `std::noreturn` (as long as all other non-discarded return statements are also of type `std::noreturn`).
On the other hand, `[] { throw; }` continues to have deduced return type `void`.

### 4.2. Library changes

`std::noreturn` is defined (in `<cstddef>`), as (the equivalent of) `decltype(throw)`.

`std::common_type` and `std::common_reference` automatically handle `std::noreturn` (via the change to the result type of the conditional expression).
Other type queries give results as above; for example, `std::is_convertible_v<std::noreturn, int>` is `true`.

The same exemption as for `void` is applied to `std::declval<cv std::noreturn>()`, such that it operates without attempting to construct an invalid reference type.

We hope it should be possible to change existing `[[noreturn]] void` functions (e.g. `std::abort`, `std::terminate`) to `std::noreturn`.  This is a user-visible API change but no-one should care too much.  It may break ABI on platforms that encode return type in name mangling, but they should be able to provide both names.

We do not propose to deprecate the attribute `[[noreturn]]`.

We also propose to add a function template, `std::invoke_noreturn`, that can be used to adapt existing presumptively `[[noreturn]]` functions to return `std::noreturn`:

```c++
template<class F, class... Args>
[[noreturn]] constexpr std::noreturn invoke_noreturn(F&& f, Args&&... args) noexcept(std::is_nothrow_invocable_v<F, Args...>) {
    std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
    std::unreachable();
}
```

## 5. Design Decisions

## 5.1 Naming
* `noreturn` / `noreturn_t`
* `empty` / `empty_t`
* `bottom` / `bottom_t`
* `⊥` / `⊥_t`
* `throw_t` (like `nullptr_t`)
* `nilstate` (like `monostate`)

## 5.2 Why not `[[noreturn]]`?

## 5.3 Why not `void`?

## 5.4 Why not an empty class type, like `std::monostate`? {#std-monostate}

## 5.5 Why not an empty class type with deleted constructors?

## 5.6 Does this deprecate [[noreturn]]?

## 5.7 Does this break API?

## 5.8 Does this break ABI?

# 6. Implementation experience

# 7. Technical specification

# 8. Acknowledgements
