---
title: Fix for `type_order` template definition
document: P3778R0
date: today
audience: LEWG, LWG
author:
  - name: Gašper Ažman
    email: <gasper.azman@gmail.com>
toc: true
---

# Abstract and problem statement

The `type_order` template is unimplementable as specified by [@P2830R10], due to
values of type `strong_ordering` not being usable as template parameters
because `strong_ordering` is not a structural type (it has a private member).

We propose replacing the _Cpp17BinaryTypeTrait with a base characteristic_
wording choice by an explicitly spelled out the struct body to avoid the issue
while preserving all possible uses.

# Design and discussion

The problematic part of [@P2830R10] wording is as follows:

> [2]{.pnum} The name `type_order` denotes a _Cpp17BinaryTypeTrait_ ([meta.rqmts]{.sref})
> with a base characteristic of `integral_constant<strong_ordering, @_TYPE-ORDER_@(@_X_@, @_Y_@)>`.

This _Cpp17BinaryTypeTrait_ concept is defined to inherit from
`integral_constant<value-type, trait-value>`.
Unfortunately, this is not possible if the intended value type is not
a structural type.

`integral_constant` is defined in [meta.help]{.sref} like so:

```cpp
template<class T, T v> struct integral_constant {
  static constexpr T value = v;

  using value_type = T;
  using type = integral_constant<T, v>;

  constexpr operator value_type() const noexcept { return value; }
  constexpr value_type operator()() const noexcept { return value; }
};
```

The specific intended uses for the design of _Cpp17BinaryTypeTrait_ are:

1. Implicit type conversion to `integral_constant<T, v>` _to hide the arguments
   to the type trait_ and thus potentially intantiate fewer templates if used
    deliberately (**cannot preserve**).
2. The embedded `::type` member to aid doing (1) explicitly (**cannot preserve**).
3. The `value` member to serve as the type trait result.
4. The `value_type` member to get the type of `value`.
5. `constexpr` implicit conversion operator to `value` for convenience.
6. `constexpr` nullary call operator to help accessing `value` from contexts
   that expect an invocable function.

We should preserve all properties we can: (3), (4), (5), and (6).

The behaviour should be as close to the rest of type traits as possible, so we
should preserve the exact behaviour of `integral_constant` in every possible
way.

We are left with the following design:

```cpp
template<class T, class U> struct type_order {
  static constexpr strong_ordering value = TYPE-ORDER(T, U);

  using value_type = strong_ordering;

  constexpr operator value_type() const noexcept { return value; }
  constexpr value_type operator()() const noexcept { return value; }
};
```

# Proposal

In [compare.type], strike paragraph 2, and fill out the code snippet in paragraph 1:

> [1]{.pnum} There is an implementation-defined total ordering of all types.
> For any (possibly incomplete) types `@_X_@` and `@_Y_@`,
> the expression `@_TYPE-ORDER_@(@_X_@, @_Y_@)` is a constant expression ([expr.const]{.sref})
> of type `strong_ordering` ([cmp.strongord]{.sref}).
> Its value is `strong_ordering::less` if `@_X_@` precedes `@_Y_@` in this
> implementation-defined total order, `strong_ordering::greater` if `@_Y_@` precedes `@_X_@`,
> and `strong_ordering::equal` if they are the same type.
> 
> [Note 1: `int`, `const int` and `int&` are different types. -- end note]
>
> [Note 2: This ordering need not be consistent with the one induced by `type_info::before`. -- end note]
>
> [Note 3: The ordering of TU-local types from different translation units is
> not observable, because the necessary specialization of `type_order` is impossible to name.
> -- end note]
> 

::: rm

> ```cpp
> template <class T, class U>
> struct type_order;
> ```

:::

::: add

>
>   ```cpp
>   template<class T, class U> struct type_order {
>     static constexpr strong_ordering value = TYPE-ORDER(T, U);
>
>     using value_type = strong_ordering;
>
>     constexpr operator value_type() const noexcept { return value; }
>     constexpr value_type operator()() const noexcept { return value; }
>   };
>   ```
>

:::

::: rm

> [2]{.pnum} The name `type_order` denotes a _Cpp17BinaryTypeTrait_ ([meta.rqmts]{.sref})
> with a base characteristic of `integral_constant<strong_ordering, @_TYPE-ORDER_@(@_X_@, @_Y_@)>`.

:::

# Notes

As of the time of this writing, `libstdc++'s` trunk implementation is as follows:

```cpp
  template<typename _Tp, typename _Up>
    struct type_order
    {
      static constexpr strong_ordering value = __builtin_type_order(_Tp, _Up);
      using value_type = strong_ordering;
      using type = type_order<_Tp, _Up>;
      constexpr operator value_type() const noexcept { return value; }
      constexpr value_type operator()() const noexcept { return value; }
    };
```

Note that it has `type`.


# Acknowledgements

The author would like to thank the following people for their contributions:

- Jakub Jelinek for implementing it in GCC and raising the alarm
- Jonathan Wakely for making sure I heard the alarm and explaining what it's about
- Barry Revzin for spelling out the obvious design above.
- Tim Song for pointing out the reason the `type` member is in `integral_constant` and that including it does us a disservice.
- Bronek Kozicki for wrangling the design space and discussion
