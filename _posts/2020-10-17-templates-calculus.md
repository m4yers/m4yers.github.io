---
layout: post
title: Templates Calculus I
description: A senseless endeavour into formalising templates calculations
comments: true
tags: [c++, templates, type-system, calculus]
---

In the previous [article]({% post_url 2020-10-16-do-cpp-templates-dream-of-lisp
%}) I have touched on the idea that you can build a Lisp-like language on top
of C++ templates and type system. Whether any Lisp is a true functional
language is a questionable question and can lead to some skirmishes. But the
templates are functional, no doubt. I was wondering if it would be possible to
build a calculus on top of templates with just types, and how far this can be
pushed.

# Basics

Lambda calculus is the smallest programming language. It was introduced by the
mathematician Alonzo Church in the 1930s as part of his research into the
foundations of mathematics. Despite not having numbers, strings, booleans, or
any non-function datatype, lambda calculus can be used to represent any Turing
Machine. Lambda calculus is composed of 3 elements: variables, functions, and
applications:

<br>

| Name        | Syntax    | Explanation                                  |
| ----------- | --------- | -------------------------------------------- |
  Variable    | *x*       | a variable named *x*
  Function    | *λx.y*    | a function with parameter *x* and body *y*
  Application | *(λx.y)a* | calling the function *λx.y* with argument *a*

<br>

In templates calculus this would look like this:

<br>

| Name        | Syntax          | Explanation                                  |
| ----------- | --------------- | -------------------------------------------- |
  Variable    | *T*             | a variable named *T*
  Function    | *template\<typename T\>*<br>*struct f { using type = U; }*     | a function with parameter *T* and body *U*
  Application | *f\<X\>::type*  | calling the function *f* with argument *X*

<br>

The most basic function is the identity function: *λx.x* which is equivalent to
*f(x) = x*. The first *x* is the function’s argument, and the second is the
body of the function. In templates this looks as following:

```cpp
template<typename T> identity { using type = T; };
```

The notions of *free* and *bound* variables in templates calculus is identical
to lambda calculus. Evaluation in templates calculus is delegated to C++
compiler templates solver.

# Currying

Lambda calculus traditionally supports only single parameter functions, we can
create multi-parameter functions using a technique called currying, which
simply points to the following property:

```
λxyz ≡ λx.λy.λz
```

Which is a fancy way of saying that a function of *N* arguments can be replaced
with a function of *1* argument returning a function of *1* argument and so
forth. C++ is somewhat rigid in its syntax, so let's start small and curry a
function of *2* arguments first:

```cpp
template<template<typename, typename> typename F>
struct curry {
  template<typename T> struct _1 {

    template<typename U> struct _2 {
      using type = typename P<T, U>::type;
    };

    template<typename U> using type = _2<U>;
  };

  template<typename T> using type = _1<T>;
};
```

What happens here is that *struct curry* declares an inner class *_1*
containing another inner class *_2*, specialising the original *T* and *U*
template arguments respectively. Notice that instead of naming those inner
types simply as *type* they have different names. This is because C++ does not
allow nested types having the same names as the enclosing types. But it is
perfectly fine with type alias having the same name. Go figure. This *struct
curry* can be used like this:

```cpp
struct Jupiter, Thebe, Io, Europa

template<class T, class U> struct product { using type = std::tuple<T, U>; };

// result is list<std::tuple<Jupiter, Thebe>,
//                std::tuple<Jupiter, Io>,
//                std::tuple<Jupiter, Europa>>
map<curry<product>::type<Jupiter>::type, list<Thebe, Io, Europa>>;
```

You can see that with currying it is possible to use all the functions from the
previous [article]({% post_url 2020-10-16-do-cpp-templates-dream-of-lisp %})
like *map* or *filter*, passing in functions of any arity, by partially
applying to them already known values. To curry functions of more than *2*
arguments you would have to create curry template structs for the required
arity, as it was shown previously.

# Arithmetic

Before we start doing arithmetic with templates calculus we need to defined
what numbers are. Recall that there are no numbers in lambda calculus, just
functions, same goes for templates calculus. But there is a way to encode
natural numbers using Church encoding with higher-order functions. The idea
behind the encoding is remarkably simple, any natural numbers is equivalent to
the number of function *f* applications, for instance this is *zero*:

```
λfz.z // same as λf.(λz.z)
````

Here, *zero* is a function that accepts another function and something else,
and evaluates to that something else. This is zero because function *f* was
applied *0* times. This is how numerals from *1* to *3* would look like:

```
λfz.f(z)        // 1, because f is applied once
λfz.f(f(z))     // 2, because f is applied twice
λfz.f(f(f(z)))  // 3, because f is applied three times
```

In templates calculus the same can be expressed the following way:

```cpp
template<template<typename> typename F, typename Z>
struct _0 { using type = Z; };

template<template<typename> typename F, typename Z>
struct _1 { using type = typename F<Z>::type; };

template<template<typename> typename F, typename Z>
struct _2 { using type = typename F<typename F<Z>::type>::type; };

template<template<typename> typename F, typename Z>
struct _3 { using type = typename F<typename F<typename F<Z>::type>::type>::type; };
```

## Successors

Defining numbers as described above is somewhat verbose in both calculi, so we
need to create a bit more machinery. What about a *successor* function, a
function that takes a number and evaluates to a number that immediately follows
it. Lambda calculus:

```
λwyx.y(wyx)
```

If you apply this function to the *zero* function you will get the following:

```
(λwyx.y(wyx))(λsz.z) = λyx.y((λsz.z)yx) = λyx.y((λz.z)x) = λyx.y(x) ≡ 1
```

And if you mentally rename *y* to *f* and *x* to *z* in the resulting function
you will see that it is equivalent to the function *1*. In templates calculus
the *successor* function is:

```cpp
template<template<template<typename> typename, typename> typename W>
struct successor {
  template<template<typename> typename Y, typename X> using type = F<W<F,Z>>;
};
```

The numerals from *0* to *3* are written this way:

```cpp
template<template<typename> typename F, typename Z> using _0 = Z;
template<template<typename> typename F, typename Z> using _1 = successor<_0>::type<F,Z>;
template<template<typename> typename F, typename Z> using _2 = successor<_1>::type<F,Z>;
template<template<typename> typename F, typename Z> using _3 = successor<_3>::type<F,Z>;
```

Verifying this result is somewhat difficult, since you need a decoder for
Church numerals to convert them into Arabic decimal numerals. You can use this
C++ counter trick:

```cpp
template<typename T> struct counter {

 using type = T;

 static constexpr int value () {
   if constexpr (std::is_same_v<void, T) { return 1; }
   else { return T::value() + 1; }
 }
};

// prints 3
std::cout << _3<counter, void>::value() << std::endl;
```

## Addition

Let's say we want to add *2* and *3*. Within our current model the result of
this computation would be invoking *successor* function *2* times over function
*3*.  In lambda calculus this expression is as follows:

```
(λfz.f(fz))(λwyx.y(wyx))(λgv.g(g(gv)))

    2        successor      3
```

By substituting all occurrences of *f* with successor and *z* with function *3*
we get:

```
(λwyx.y(wyx)) ( (λwyx.y(wyx)) (λgv.g(g(gv))) )

   successor      successor         3
```

Unfortunately the following won't work in C++, because a templates calculus
numeral expects a function of one *template template* argument, but *successor*
single argument is of much more complex template layout:

```cpp
// error: template template argument has different template parameters
          than its corresponding template template parameter
_2<successor, _3>;
```

Luckily there is another way. If you look at the numerals' signatures you will
see that the second parameter is called *Z* and it kinda means *zero*, so if
you place other numeral there you would get addition:

```cpp
template<template<typename> typename F, typename Z> using _5 = _2<F, _3<F,Z>>;
```

To generalise a bit more:

```cpp
template<template<template<typename> typename, typename> typename L,
         template<template<typename> typename, typename> typename R>
struct add {
  template<template<typename> typename F, typename Z> using type = L<F, R<F,Z>>;
};

template<template<typename> typename F, typename Z> using _5 = add<_2,_3>::type<F,Z>;
```

## Multiplication

Multiplication can be done in the same vein as the addition:

```cpp
template<template<template<typename> typename, typename> typename L,
         template<template<typename> typename, typename> typename R>
struct multiply {
  template<template<typename> typename F, typename Z>
  using type = L<R, Z>;
};
```

But this won't compile because numerals expect two arguments, and the first
argument is a function of one argument, which means you cannot simply pass
numeral to a numeral as argument. This is where we need to use currying, though
the function described above won't fit here, we need to make it work for
*template template* parameters:

```cpp
template<template<template<typename> typename, typename> typename F>
struct curry {
  template<template<typename> typename T> struct _1 {

    template<typename U> struct _2 {
      using type = F<T, U>;
    };

    template<typename U> using type = typename _2<U>::type;
  };

  template<template<typename> typename T> using type = _1<T>;
};
```

And finally multiplication looks like this:

```cpp
template<template<template<typename> typename, typename> typename L,
         template<template<typename> typename, typename> typename R>
struct multiply {
  template<template<typename> typename F, typename Z>
  using type = L<curry<R>::template type<F>::template type, Z>;
};

template<template<typename> typename F, typename Z>
using _21 = multiply<_3, _7>::type<F, Z>;
```

Phew, that's a lot of templates. Next time I might elaborate on conditionals,
recursions, data-structures and so forth. Till the next time, cheers!
