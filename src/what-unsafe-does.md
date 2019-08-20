# What Unsafe Rust Can Do

The only things that are different in Unsafe Rust are that you can:

* Dereference raw pointers
* Call `unsafe` functions (including C functions, compiler intrinsics, and the raw allocator)
* Implement `unsafe` traits
* Mutate statics
* Access fields of `union`s

That's it. The reason these operations are relegated to Unsafe is that misusing
any of these things will cause the ever dreaded Undefined Behavior. Invoking
Undefined Behavior gives the compiler full rights to do arbitrarily bad things
to your program. You definitely *should not* invoke Undefined Behavior.



## Fundamental Undefined Behaviour

Unlike C, Undefined Behavior is pretty limited in scope in Rust. All the core
language cares about is preventing the following things:

* Dereferencing (using the `*` operator on) a raw pointer that is dangling, unaligned, or that has invalid metadata (if wide; see references below)
* Breaking the [pointer aliasing rules][]
* Unwinding out of a function that doesn't have a rust-native [calling convention][]
* Causing a [data race][race]
* Executing code compiled with [target features][] that the current thread of execution does
  not support
* Producing invalid values (either alone or as a field of a compound type such
  as `enum`/`struct`/array/tuple):
    * a `bool` that isn't 0 or 1
    * an `enum` with an invalid discriminant
    * a `fn` pointer that is null
    * a `char` outside the ranges [0x0, 0xD7FF] and [0xE000, 0x10FFFF]
    * a `!` (all values are invalid for this type)
    * a reference that is dangling, unaligned, points to an invalid value, or
      that has invalid metadata (if wide)
        * slice metadata is invalid if the slice has a total size larger than
          `isize::MAX` bytes in memory
        * `dyn Trait` metadata is invalid if it is not a pointer to a vtable for
          `Trait` that matches the actual dynamic trait the reference points to
    * a `str` that isn't valid UTF-8
    * a non-padding byte that is [uninitialized memory][] (see discussion below)
    * a type with custom invalid values that is one of those values, such as a
      `NonNull` that is null. (Requesting custom invalid values is an unstable
      feature, but some stable stdlib types, like `NonNull`, make use of it.)

A reference/pointer is "dangling" if it is null or not all of the bytes it
points to are part of the same allocation (so in particular they all have to be
part of *some* allocation). The span of bytes it points to is determined by the
pointer value and the size of the pointee type. As a consequence, if the span is
empty, "dangling" is the same as "non-null". Note that slices point to their
entire range, so it's very important that the length metadata is never too
large. If for some reason this is too cumbersome, consider using raw pointers.



## Invalid Values: Yes We Mean It

Many have trouble accepting the consequences of invalid values, so they merit
some extra discussion here so no one misses it. The claim being made here is a
very strong and surprising one, so read carefully.

A value is *produced* whenever it is assigned, passed to something, or returned
from something. Keep in mind references get to assume their referents are valid,
so you can't even create a reference to an invalid value.

Additionally, [uninitialized memory][] is **always invalid**, so you can't assign it to
anything, pass it to anything, return it from anything, or take a reference to it.
Padding bytes aren't technically part of a value's memory, and so may be left
uninitialized. For unions, this includes the padding bytes of *all* variants,
as unlike enums, unions are never definitely set to any particular variant (Rust
does not have the C++ notion of an "active member"). This makes unions
are the preferred mechanism for working directly with uninitialized memory (see
[MaybeUninit][] for details).

In simple and blunt terms: you cannot ever even *suggest* the existence of an
invalid value. No, it's not ok if you "don't use" or "don't read" the value.
Invalid values are **instant Undefined Behaviour**. The only correct way to
manipulate memory that could be invalid is with raw pointers using methods
like write and copy. If you want to leave a local variable or struct field
uninitialized (or otherwise invalid), you must use a union (like MaybeUninit)
or enum (like Option) which clearly indicates at the type level that this
memory may not be part of any value.




## Other Sources of Undefined Behavior

That's it. That's all the causes of Undefined Behavior baked into Rust.

Well, ok, only sort of.

While it's true that the language itself doesn't define that much Undefined
Behavior, libraries may use unsafe functions and unsafe traits to define
their own contracts with Undefined Behavior at stake. For instance, the raw
allocator APIs declare that you aren't allowed to deallocate unallocated memory,
and the Send trait declares that implementors must in fact be safe to move to
another thread.

Usually these constraints are in place because violating them will lead to one
of Rust's Fundamental Undefined Behaviors, but that doesn't have to be the case.
In particular, several standard library APIs are actually thin wrappers around
*intrinsics* which tell the compiler it can make certain assumptions.

It's useful to distinguish between these "intrinsic" sources of UB and
the fundamental ones because the intrinsic ones *don't matter* unless someone
actually invokes the relevant functions. The fundamental ones, on the other hand,
are ever-present.

With that said, some intrinsics, like the surprisingly strict [`ptr::offset`][],
are *pretty* close to fundamental. 😅



## Not Technically Fundamental Undefined Behavior

There are a few things in Rust that aren't *technically* Fundamental Undefined Behavior,
but which library authors can implicitly assume don't happen, with Undefined
Behavior at stake. As such, it should be impossible to do these things in safe
code, as they can very easily lead to Undefined Behavior.

This section is non-exhaustive, although that may change in the future.

It is *technically not* Undefined Behavior to run a value's destructor twice.
Authors of destructors may however assume this doesn't happen. For instance, if
you drop a Box twice it will almost certainly result in Undefined Behavior.
Technically someone *could* explicitly support double-dropping their type, although
it's hard to say why.

It is *technically not* Undefined Behavior to reinterpret a bunch of
bytes as a type whose fields you don't have public access to (assuming you
don't create any Invalid Values). As [the next section][] discusses, it's very
important for library authors to be able to rely on privacy and ownership as a
sort of program integrity proof. For instance, if you reinterpret some random
non-zero bytes as a Vec, this will almost certainly result in Undefined Behavior.
It's very important that you *can* just create types from a bunch of bytes if
done correctly (such as pairing ptr::read with ptr::write).




## Completely Safe Behavior

Rust can also be quite permissive of dubious operations.
Rust considers it "safe" to:

* Deadlock
* Have a [race condition][race]
* Leak memory
* Fail to call destructors
* Overflow integers
* Abort the program
* Delete the production database

However any program that actually manages to do such a thing is *probably*
incorrect. Rust provides lots of tools to make these things rare, but
some things are just impractical to categorically prevent.

[pointer aliasing rules]: references.html
[uninitialized memory]: uninitialized.html
[the next section]: working-with-unsafe.html
[race]: races.html
[target features]: ../reference/attributes/codegen.html#the-target_feature-attribute
[MaybeUninit]: ../core/mem/union.MaybeUninit.html
[calling convention]: ../reference/items/external-blocks.html#abi
[`ptr::offset`]: ../core/primitive.pointer.html#method.offset
