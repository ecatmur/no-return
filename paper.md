# Scraping the `[[noreturn]]`: unreachability in the type system

## 1. Abstract
```
using noreturn_t = decltype(throw);
```
We already have `[[noreturn]]` as an annotation[^1] and P0627[^2] will give us `std::unreachable`.
This paper proposes adding unreachability to the type system in a complementary manner.

## 2. Motivation
`std::visit` requires its visitor to return "a valid expression of the same type and value category, for all combinations of alternative types of all variants"[^3].
A more advanced visit function might deduce its return type using `std::common_reference`:

```
template<class, class, class> struct common_result_impl;
template<class Vis, class Var, size_t... I>
struct common_result_impl<Vis, Var, index_sequence<I...>> : common_reference<decltype(declval<Vis>()(get<I>(declval<Var>())))...> {};
template<class Vis, class Var> using common_result = common_result_impl<Vis, Var, make_index_sequence<variant_size_v<remove_cvref_t<Var>>>>;
template<class Vis, class Var> typename common_result<Vis, Var>::type better_visit(Vis&& vis, Var&& var) {
    return visit([&]<class Arg>(Arg&& arg) -> typename common_result<Vis, Var>::type { return forward<Vis>(vis)(forward<Arg>(arg)); }, var);
}
int i = better_visit([](auto x) { return x; }, variant<int, char, long>('a'));
```

However this falls apart if the visitor is inapplicable to some alternatives; the user may throw for particular alternatives, or may have ensured ahead of time that the variant does not contain particular alternatives, so can mark those code paths unreachable - but cannot communicate this to its caller, since all it can do is mark itself `[[noreturn]]`, which is not visible in the type system.

## 3. Proposal
We propose a type to denote unreachability; in type theory this is variously called the "bottom" type[^4] (since it sits at the bottom of the type heirarchy) or the "empty" type (since it has no values).

This will be defined as the type of a throw expression[^5] (as `nullptr_t` is the type of `nullptr`).

### 3.1. Language changes
`decltype(throw)` will be valid and yield the type `noreturn_t`.

For forming compound types, the same restrictions apply as with `void`: no references, arrays, etc.

A program that creates a value of type `noreturn_t` in evaluated context (including constant evaluation) is ill-formed.

In a conditional expression, if either branch is of type `noreturn_t`, the result is the type and category of the other branch. This replaces the existing language on throw expressions.

Like `void`[^6], it it is allowed to return values of type `noreturn_t` from function templates, but otherwise values of type `noreturn_t` cannot be created or stored (whether in a variable or data member).

There are no changes proposed to return type deduction; a function template may use `return throw [expression];` to deduce to `noreturn_t` (as long as all other non-discarded return statements are also of type `noreturn_t`).

### 3.2. Library changes
`noreturn_t`[^7] is defined (in `<cstddef>`), as (the equivalent of) `decltype(throw)`.

`common_type` and `common_reference` automatically handle `noreturn_t` (via the change to the result type of the conditional expression).

We hope it should be possible to make existing `[[noreturn]] void` functions (e.g. `std::abort`, `std::terminate`) to `noreturn_t`.  This is a user-visible API change but no-one should care too much.  It may break ABI on platforms that encode return type in name mangling, but they should be able to provide both names.

We do not propose to deprecate the attribute `[[noreturn]]`.

## 4. Bikeshedding
* `noreturn` / `noreturn_t`
* `empty` / `empty_t`
* `bottom` / `bottom_t`
* `⊥` / `⊥_t`
* `throw_t` (like `nullptr_t`)
* `nilstate` (like `monostate`)

## Z. Footnotes

[^1]: XXX find when `[[noreturn]]` was added
[^2]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p0627r6.pdf
[^3]: Link eel.is or some such
[^4]: Wikipedia
[^5]: It does not depend on the argument to the throw expression (if there is one); `throw` erases the type of its operand, and `catch` restores it
[^6]: Assuming "regular void" isn't on the cards
[^7]: See section Bikeshedding for possible names
