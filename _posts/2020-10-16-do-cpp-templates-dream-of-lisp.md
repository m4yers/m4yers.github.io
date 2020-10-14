---
layout: post
title: Do C++ templates dream of Lisp?
description: I semi-practical experiment of merging C++ templates and lisp
comments: true
tags: [c++, templates, type-system, lisp]
---

Imagine a Lisp embedded in C++. What kind of possibilities this would unlock?
Fluid syntax and semantics, code is data and data is code. Read–eval–print loop
for fast, iterative development. An omnipotent macro system to push the
language to its limits. When you program in this environment, it is like you
are talking directly to compiler. All your S-expressions neatly translate
into AST. All the power of Lisp is under your fingertips! Then, why not just
use Lisp, without C++?..

There is a reason why C++ is the language of choice for many software
engineers, and not Lisp. They want *speed*. They want to feel the wind blowing
through their hair, how fast a program written in C++ can run. There is
absolutely no competition here for C++, in terms of performance and
expressiveness. There is no need even to look in the direction of Lisp for a
C++ engineer. Unless you consider templates.

Hardly anyone sees that dormant functional power in the meta language of the
wicked. The templates can indeed be considered a separate language from C++,
evaluated at compile-time, instead of runtime. They have no side-effects. Any
function you write with templates is pure. This language of course lacks *io*
to make it useful, it is *too* pure. But the main drawback, many people would
agree, is its expressiveness, or rather the lack of it. One way to fix this is
to fork a C++ compiler, for instance Clang, and modify the templates syntax,
and maybe semantics, to your taste. This could work, except that this is not
very portable. Only you and a couple of your friends can enjoy this new C++.
There is another path, and this path is full of lists.

So, C++ templates and Lisp, let's see. The basic building blocks of any lisp is
two things: atoms and lists. Templates mainly operate on types, so for
simplicity sake let us use type atoms. The list then looks like this:

```
template<typename... Ts> struct list {};
```

To manipulate lists we need some functions. The most basic LISP functions you
can find are *car* and *cdr*. With a bit of renaming we have the following:

```
// first
template<typename T, typename... Ts>
constexpr auto first_impl(list<T, Ts...>) -> T;

template<typename L>
using first = decltype(first_impl(L{}));

// tail
template<typename T, typename... Ts>
constexpr auto tail_impl(list<T, Ts...>) -> list<Ts...>;

template<typename L>
using tail = decltype(tail_impl<>(L{}));
```

With only these two functions you already can freely manipulate lists of types.
How about type maps:

```
// map
template<template<typename> typename F, typename... Ts>
constexpr auto map_impl(list<Ts...>) -> list<typename F<Ts>::type...>;

template<template<typename> typename F, typename L>
using map = decltype(map_impl<F>(L{}));
```

This one is a bit tougher to digest, but the semantics is quite
straightforward, this function applies a function *F* over all elements of list
*L* and returns a new list with the results. For example to *const* all types
you would do the following:

```
struct Deckard, Batty, Bryant, Chew, Gaff;

// result is list<const Deckard, const Batty, const Bryant, const Chew, const Gaff>
map<std::add_const, list<Deckard, Batty, Bryant, Chew, Gaff>>;
```

Existing C++ type traits are perfectly fine with such treatment. Now you can
map any trait over any list of types in one go. In the same vein we can
implement filtering:

```
// tuple
template<typename... Ts> constexpr
bool is_tuple = false;

template<typename... Ts> constexpr
bool is_tuple<std::tuple<Ts...>> = true;

template<typename... Ts>
constexpr auto to_list_impl(std::tuple<Ts...>) -> list<Ts...>;

template<typename T, typename = std::enable_if_t<is_tuple<T>>>
using to_list = decltype(to_list_impl(T{}));

// filter
template<template<typename> typename P, typename... Ts>
constexpr auto filter_impl(list<Ts...>)
  -> to_list<decltype(std::tuple_cat(std::conditional_t<P<Ts>::value,
                                     std::tuple<Ts>,
                                     std::tuple<>>{}...))>;

template<template<typename> typename P, typename L>
using filter = decltype(filter_impl<P>(L{}));
```

To save some space and time we used *std::tuple* to deal with *holes* in the
output list. Nevertheless, the resulting function allows us once again use some
of the C++ type traits to filter list of types:

```
struct Holden, Leon, Taffey;

// result is list<const Leon>
filter<std::is_const, list<Holden, const Leon, Taffey>>;
```

In this example we deviate from the classic predicate implementation. In Lisp,
predicates are functions that test their arguments for some specific conditions
and returns *nil* if the condition is *false*, or some *non-nil* value is the
condition is *true*. In C++ type trait predicates do not work like that, they
are usually implemented as a type wrapper around a *bool* value. This is
perfectly fine for filtering, but if you want to use these predicates with the
*map* function, you would want to consider wrapping those bools in *nil* and
*non-nil*. Nil by the way can be implemented like this:

```
using nil = list<>
```

With these basics techniques you can already start building up your own
templates lisp, and try writing more complex type transformations. This comes
in handy when you have to create elaborate data-structures, like maps of maps
of tuples of vectors with a specific layout. This way your data-structure
layout becomes a function, and the arguments to this function are map key
types, value types and so on.

That's all for today folks. In the following articles I will touch more on
functional programming with C++ templates. Stay tuned.
