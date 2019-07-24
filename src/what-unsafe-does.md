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

Unlike C, Undefined Behavior is pretty limited in scope in Rust. All the core
language cares about is preventing the following things:

* Dereferencing (using the `*` operator on) null, dangling, or unaligned
  pointers, or fat pointers with invalid metadata (see below)
* Reading [uninitialized memory][]
* Breaking the [pointer aliasing rules][]
* Producing invalid primitive values (either alone or as a field of a compound
  type such as `enum`/`struct`/array/tuple):
    * a `bool` that isn't 0 or 1
    * an undefined `enum` discriminant
    * null `fn` pointers
    * a `char` outside the ranges [0x0, 0xD7FF] and [0xE000, 0x10FFFF]
    * a `!` (all values are invalid for this type)
    * dangling/null/unaligned references, references that do themselves point to
      invalid values, or fat references (to a dynamically sized type) with
      invalid metadata
        * slice metadata is invalid if the slice has a total size larger than
          `isize::MAX` bytes in memory
        * `dyn Trait` metadata is invalid if it is not a pointer to a vtable for
          `Trait` that matches the actual dynamic trait the reference points to
    * a non-utf8 `str`
    * an uninitialized integer (`i*`/`u*`) or floating point value (`f*`)
    * an invalid library type with custom invalid values, such as a `NonNull` or
      `NonZero*` that is 0
* Unwinding into another language
* Causing a [data race][race]
* Executing code compiled with platform features that the current platform does
  not support (see [`target_feature`])

"Producing" a value happens any time a value is assigned, passed to a
function/primitive operation or returned from a function/primitive operation.

A reference/pointer is "dangling" if not all of the bytes it points to are part
of the same allocation. The span of bytes it points to is determined by the
pointer value and the size of the pointee type.

That's it. That's all the causes of Undefined Behavior baked into Rust. Of
course, unsafe functions and traits are free to declare arbitrary other
constraints that a program must maintain to avoid Undefined Behavior. For
instance, the allocator APIs declare that deallocating unallocated memory is
Undefined Behavior.

However, violations of these constraints generally will just transitively lead to one of
the above problems. Some additional constraints may also derive from compiler
intrinsics that make special assumptions about how code can be optimized. For instance,
Vec and Box make use of intrinsics that require their pointers to be non-null at all times.

Rust is otherwise quite permissive with respect to other dubious operations.
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
these problems are considered impractical to categorically prevent.

[pointer aliasing rules]: references.html
[uninitialized memory]: uninitialized.html
[race]: races.html
[`target_feature`]: ../reference/attributes/codegen.html#the-target_feature-attribute
