# visit_struct

[![Build Status](https://travis-ci.org/cbeck88/visit_struct.svg?branch=master)](http://travis-ci.org/cbeck88/visit_struct)
[![Appveyor status](https://ci.appveyor.com/api/projects/status/6ksqg7es938cttn2/branch/master?svg=true)](https://ci.appveyor.com/project/cbeck88/visit_struct)
[![Boost licensed](https://img.shields.io/badge/license-Boost-blue.svg)](./LICENSE)

A header-only library providing **structure visitors** for C++11.

## Motivation

In C++ there is no built-in way to iterate over the members of a `struct` type.

Oftentimes, an application may contain several small "POD" datatypes, and one
would like to be able to easily serialize and deserialize, print them in debugging
info, and so on. Usually, the programmer has to write a bunch of boilerplate
for each one of these, listing the struct members over and over again.

(This is only the most obvious use of structure visitors.)

Naively one would like to be able to write something like:

```c++
for (const auto & member : my_struct) {
  std::cerr << member.name << ": " << member.value << std::endl;
}
```

However, this syntax can never be legal in C++, because when we iterate using a
for loop, the iterator has a fixed static type, and `member.value` similarly has
a fixed static type. But the struct member types must be allowed to vary.

## Visitors

The usual way to overcome issues like that (without taking a performance hit)
is to use the *visitor pattern*. For our purposes, a *visitor* is a generic callable
object. Suppose our struct looks like this:

```c++
struct my_type {
  int a;
  float b;
  std::string c;
};
```

and suppose we had a function like this, which calls the visitor `v` once for
each member of the struct:

```c++
template <typename V>
void visit(V && v, const my_type & my_struct) {
  v("a", my_struct.a);
  v("b", my_struct.b);
  v("c", my_struct.c);
}
```

(For comparison, see also the function `boost::apply_visitor` from the `boost::variant` library,
which similarly applies a visitor to the value stored within a variant.)

Then we can "simulate" the for-loop that we wanted to write in a variety of ways. For instance, we can
make a template function out of the body
of the for-loop and use that as a visitor.

```c++
template <typename T>
void log_func(const char * name, const T & value) {
  std::cerr << name << ": " << value << std::endl;
}

visit(log_func, my_struct);
```

Using a template function here means that even though a struct may contain several different types, the compiler
figures out which function to call at compile-time, and we don't do any run-time polymorphism -- the whole call can
often be inlined.

Basically we are solving the original problem in a very exact way -- there is no longer an explicit
iterator, and each time the "loop body" can be instantiated with different types as needed.

If the loop has internal state or "output", we can use a function object (an object which overloads `operator()`) as the visitor,
and collect the state in its members. Also in C++14 we have generic lambdas, which sometimes makes all this very terse.

Additionally, while making a visitor is sometimes more verbose than you'd like, it has an added benefit that generic visitors can
be used and reused many times. Often, when doing things like logging or serialization, you don't want each struct to get a different
implementation or policy, you want to reuse the same code for all of them.

## Reflection

So, if we have a template function `visit` for our struct, it may let us simplify code and promote code reuse.

However, that means we still have to actually define `visit` for every struct we want to use it
with, and possibly several versions of it, taking `const my_type &`, `my_type &`, `my_type &&`, and so on.
That's also quite a bit of repetitive code, and the whole point of this is to reduce repetition.

Again, ideally we would be able to do something totally generic, like,

```c++
template <typename V, typename S>
void apply_visitor(V && v, S && s) {
  for (auto && member : s) {
    v(member.name, member.value);
  }
}
```

where both the visitor and struct are template parameters, and use this to visit the members of any struct.

Unfortunately, current versions of C++ *lack reflection*. It's not possible
to *programmatically inspect* the list of members of a generic class type `S`, using templates or
anything else standard, even if `S` is a complete type (in which case, the compiler obviously
knows its members). If we're lucky we might get something like this in C++20, but right
now there's no way to actually implement the fully generic `apply_visitor`.

## Overview

This library permits the following syntax in a C++11 program:

```c++
struct my_type {
  int a;
  float b;
  std::string c;
};

VISITABLE_STRUCT(my_type, a, b, c);



struct debug_printer {
  template <typename T>
  void operator()(const char * name, const T & value) {
    std::cerr << name << ": " << value << std::endl;
  }
};

void debug_print(const my_type & my_struct) {
  visit_struct::apply_visitor(debug_printer{}, my_struct);
}
```

Here, the macro `VISITABLE_STRUCT` defines overloads of `visit_struct::apply_visitor`
for your structure.

In C++14 this can be made more succinct using a lambda:

```c++
const auto debug_printer = [](const char * name, const auto & value) {
  std::cerr << name << ": " << value << std::endl;
};

void debug_print(const my_type & my_struct) {
  visit_struct::apply_visitor(debug_printer, my_struct);
}
```

These two things, the macro `VISITABLE_STRUCT` and the function `visit_struct::apply_visitor`,
are basically the whole library.

A nice feature of `visit_struct` is that `apply_visitor` always respects the
C++11 value category of it's arguments.
That is, if `my_struct` is a const l-value reference, non-const l-value reference, or r-value
reference, then `apply_visitor` will pass each of the fields to the visitor correspondingly,
and the visitor is also forwarded properly.

It should be noted that there are already libraries that permit iterating over a structure like
this, such as `boost::fusion`, which does this and much more. Or `boost::hana`, which is like
a more modern successor to `boost::fusion` which takes advantage of C++14.

However, our library can be used as a single-header, header-only library with no external dependencies.
The core `visit_struct.hpp` is in total about three hundred lines of code, depending on how you count,
and is fully functional on its own. For some applications, `visit_struct` is all that you need.

**Note:** The macro `VISITABLE_STRUCT` must be used at filescope, an error will occur if it is
used within a namespace. You can simply include the namespaces as part of the type, e.g.

```c++
VISITABLE_STRUCT(foo::bar::baz, a, b, c);
```

## Compatibility with `boost::fusion`

**visit_struct** also has support code so that it can be used with "fusion-adapted structures".
That is, any structure that `boost::fusion` knows about, can also be used with `visit_struct::apply_visitor`,
if you include the extra header.  

`#include <visit_struct/visit_struct_boost_fusion.hpp>`

This compatability header means that you don't have to register a struct once with `fusion` and once with `visit_struct`.
It may help if you are migrating from one library to the other.

## Compatiblity with `boost::hana`

**visit_struct** also has a similar compatibility header for `boost::hana`.  

`#include <visit_struct/visit_struct_boost_hana.hpp>`

## "Intrusive" Syntax

An additional header is provided, `visit_struct_intrusive.hpp` which permits the following alternate syntax:

```c++
struct my_type {
  BEGIN_VISITABLES(my_type);
  VISITABLE(int, a);
  VISITABLE(float, b);
  VISITABLE(std::string, c);
  END_VISITABLES;
};
```

This declares a structure which is essentially the same as

```c++
struct my_type {
  int a;
  float b;
  std::string c;
};
```

There are no additional data members defined within the type, although there are
some "secret" static declarations which are occurring. That's why it's "intrusive".
There is still no run-time overhead.

Each line above expands to a separate series of declarations within the body of `my_type`, and arbitrary other C++
declarations may appear between them.

```c++
struct my_type {

  int not_visitable;
  double not_visitable_either;

  BEGIN_VISITABLES(my_type);
  VISITABLE(int, a);
  VISITABLE(float, b);

  typedef std::pair<std::string, std::string> spair;

  VISITABLE(spair, p);

  void do_nothing() const { }

  VISITABLE(std::string, c);

  END_VISITABLES;
};
```

When `visit_struct::apply_visitor` is used, each member declared with `VISITABLE`
will be visited, in the order that they are declared.

The benefits of this version are that, you don't need to type all the member
names twice, and you don't need to jump out of your namespaces back to filescope
in order to register a struct. The main drawbacks are that this is still somewhat
verbose, the implementation is a bit more complicated, and this one may not be
useful in some cases, like if the struct you want to visit belongs to some other
project and you can't change its definition.

## Visitation without an instance

Besides iteration over an *instance* of a registered struct, **visit_struct** also
supports visiting the *definition* of the struct. In this case, instead of passing
you the field name and the field value within some instance, it passes you the
field name and the *pointer to member* corresponding to that field.

For instance, the function call

```c++
visit_struct::apply_visitor<my_type>(v);
```

is similar to

```c++
v("a", &my_type::a);
v("b", &my_type::b);
v("c", &my_type::c);
```

This is potentially very useful in some situations. For instance, sometimes you want to
output a diagnostic about the layout of some object, but actually instantiating
it is complicated or expensive. With this version of `apply_visitor`, you get the names
and the types of the members, without needing to actually instantiate the object.

This may be especially useful when you have a C++14 compiler which has proper `constexpr` support.
In that case, `visit_struct::apply_visitor` is `constexpr` also, so you can use this
for some nifty metaprogramming purposes. (For an example, check out [test_fully_visitable.cpp](./test_fully_visitable.cpp).)

Much thanks to Jarod42 for this patch.




**Note:** The compatibility headers for `boost::fusion` and `boost::hana` don't
currently support this version of `apply_visitor`.

I don't know how to get the pointers-to-members
like this from `boost::fusion` -- in [this stackoverflow answer](http://stackoverflow.com/questions/35893937/pointers-to-class-members-when-iterating-with-boostfusion) user `jv_`
says he doesn't believe it's possible to get those pointers which `fusion` holds internally, and the smart bet is that he's right about such things.

In the case of `hana`, it's not likely to be able to get them
because it goes somewhat against the design, which views the "`struct` concept" as essentially "sequences of move-invariant values". Internally it represents all structs as tuples, and attempts to abstract away details like pointers to members. See the `hana` documentation for more on this.

If you really want or need to be able to get the pointers to members, that's a pretty good reason to use `visit_struct` honestly.

## Binary Vistation

**visit_struct** also supports visiting two instances of the same struct at once.

The syntax is similar to that of `boost::variant` -- the visitor comes first,
then two the structure instances.

For instance, the function call

```c++
visit_struct::apply_visitor(v, s1, s2);
```

is similar to

```c++
v("a", s1.a, s2.a);
v("b", s1.b, s2.b);
v("c", s1.c, s2.c);
```

This is useful for implementing generic equality and comparison operators for visitable
structures, for instance. Here's an example of a generic function `struct_eq` which
compares any two visitable structures for equality using `operator ==` on each field,
and which short-circuits properly.

```c++
struct eq_visitor {
  bool result = true;

  template <typename T>
  void operator()(const char *, const T & t1, const T & t2) {
    result = result && (t1 == t2);
  }
};

template <typename T>
bool struct_eq(const T & t1, const T & t2) {
  eq_visitor vis;
  visit_struct::apply_visitor(vis, t1, t2);
  return vis.result;
}
```

## Tuple Methods

For a long time, `apply_visitor` was the only function in the library. It's quite
powerful and everything that I needed to do could be done using it comfortably.
I like having a really small API.

However, one thing that you cannot easily use `apply_visitor` to do is implement `std::tuple`
methods, like `std::get<i>` to get the `i`'th member of the struct. By contrast, `apply_visitor`
could be implemented with relative ease in C++11 using `std::get` and variadic templates,
so arguably this is a more fundamental operation. Indeed, most if not all libraries that support struct-field reflection support this in some way.
So, we decided it was remiss not to support this also.

We didn't change our implementation of `apply_visitor`, which works well on all targets right now including MSVC 2013.
But we have added three new functions which allow indexed access to structures.

```c++
visit_struct::get<i>(struct);
```

Gets (a reference to) the `i`'th visitable element of the struct. Index is 0-based. Analogous to `std::get`.

```c++
visit_struct::get_name<i>(struct);
```

Gets a `const char *` pointing to the name of the `i`'th visitable element of the struct. The struct type may also be passed as a second template parameter
instead of as an argument.

```c++
visit_struct::field_count<S>();
```

Gets a `size_t` which tells how many visitable fields there are.


## Limits

When using `VISITABLE_STRUCT`, the maximum number of members which can be registered
is `visit_struct::max_visitable_members`, which is by default 69.

When using the intrusive syntax, the maximum number of members is `visit_struct::max_visitable_members_intrusive`,
which is by default 100.

These limits can both be increased, see the source comments and also [IMPLEMENTATION_NOTES.md](/IMPLEMENTATION_NOTES.md).

## Compiler Support

**visit_struct** targets C++11 -- you need to have r-value references at least, and for the intrusive syntax, you need
variadic templates also.

**visit_struct** works with versions of gcc `>= 4.8.2` and versions of clang `>= 3.5`.

The appveyor build tests against MSVC 2013, 2015, 2017.

MSVC 2015 is believed to be fully supported.

For MSVC 2013, the basic syntax is supported, the intrusive syntax doesn't work there and now isn't tested.
Again, patches welcome.

Much thanks again to Jarod42 for significant patches related to MSVC support.

## Constexpr Correctness

`visit_struct` attempts to target three different levels of `constexpr` support.

- No support
- C++11 support
- C++14 extended support

This is controlled by two macros `VISIT_STRUCT_CONSTEXPR` and `VISIT_STRUCT_CXX14_CONSTEXPR`. We use these tokens where we would use the `constexpr` keyword.

In the `visit_struct.hpp` header, these macros are defined to either `constexpr` or nothing.

We attempt to guess the appropriate setting by inspecting the preprocessor symbols `__cplusplus` and `_MSC_VER`.

If it doesn't work on your compiler, please open a github issue, especially if you know how to fix it :)

In the meantime, if you don't want to tweak the headers for your project, you can override the behavior by defining these macros yourself, before including `visit_struct.hpp`.
If the header sees that you have defined them it won't touch them and will defer to your settings. In most cases this should not be necessary.

On gcc and clang, we assume at least C++11 constexpr support. If you enabled a later standard using `-std=...`, we turn on the full `constexpr`.

On MSVC currently the settings are:

- VS2013: no support
- VS2015: C++11 support
- VS2017: C++14 extended support

## Licensing and Distribution

**visit_struct** is available under the boost software license.

## See also

* [map-macro](https://github.com/swansontec/map-macro) from swansontec
* [boost-hana](http://www.boost.org/doc/libs/1_61_0/libs/hana/doc/html/index.html) from ldionne
* [pod flat reflection](https://github.com/apolukhin/magic_get) from apolukhin
* [self-aware structs](http://duriansoftware.com/joe/Self-aware-struct-like-types-in-C++11.html) from jckarter
