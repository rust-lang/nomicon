# Alternative representations

Rust allows you to specify alternative data layout strategies from the default.




# repr(C)

This is the most important `repr`. It has fairly simple intent: do what C does.
The order, size, and alignment of fields is exactly what you would expect from C
or C++. Any type you expect to pass through an FFI boundary should have
`repr(C)`, as C is the lingua-franca of the programming world. This is also
necessary to soundly do more elaborate tricks with data layout such as
reinterpreting values as a different type.

However, the interaction with Rust's more exotic data layout features must be
kept in mind. Due to its dual purpose as "for FFI" and "for layout control",
`repr(C)` can be applied to types that will be nonsensical or problematic if
passed through the FFI boundary.

* ZSTs are still zero-sized, even though this is not a standard behavior in
C, and is explicitly contrary to the behavior of an empty type in C++, which
still consumes a byte of space.

* DST pointers (fat pointers), tuples, and enums with data (that is, non-C-like
  enums) are not a concept in C, and as such are never FFI-safe.

* As an exception to the rule on enum with data, `Option<&T>` is
  FFI-safe if `*const T` is FFI-safe, and they have the same
  representation, using the null-pointer optimization: `Some<&T>` is
  represented as the pointer, and `None` is represented as a null
  pointer.  (This rule applies to any enum defined the same way as
  libcore's `Option` type, and using the default repr.)

* Tuple structs are like structs with regards to `repr(C)`, as the only
  difference from a struct is that the fields aren’t named.

* **If the type would have any [drop flags], they will still be added**

* This is equivalent to one of `repr(u*)` (see the next section) for enums. The
chosen size is the default enum size for the target platform's C ABI. Note that
enum representation in C is implementation defined, so this is really a "best
guess". In particular, this may be incorrect when the C code of interest is
compiled with certain flags.

* "C-like" enums with `repr(C)` or `repr(u*)` still may not be set to an
integer value without a corresponding variant, even though this is
permitted behavior in C or C++. It is undefined behavior to (unsafely)
construct an instance of an enum that does not match one of its
variants. (This allows exhaustive matches to continue to be written and
compiled as normal.)



# repr(u8), repr(u16), repr(u32), repr(u64)

These specify the size to make a C-like enum. If the discriminant overflows the
integer it has to fit in, it will produce a compile-time error. You can manually
ask Rust to allow this by setting the overflowing element to explicitly be 0.
However Rust will not allow you to create an enum where two variants have the
same discriminant.

The term "C-like enum" only means that the enum doesn't have data in any
of its variants. A C-like enum without a `repr(u*)` or `repr(C)` is
still a Rust native type, and does not have a stable ABI representation.
Adding a `repr` causes it to be treated exactly like the specified
integer size for ABI purposes.

A non-C-like enum is a Rust type with no guaranteed ABI (even if the
only data is `PhantomData` or something else with zero size).

Adding an explicit `repr` to an enum suppresses the null-pointer
optimization.

These reprs have no effect on a struct.




# repr(packed)

`repr(packed)` forces Rust to strip any padding, and only align the type to a
byte. This may improve the memory footprint, but will likely have other negative
side-effects.

In particular, most architectures *strongly* prefer values to be aligned. This
may mean the unaligned loads are penalized (x86), or even fault (some ARM
chips). For simple cases like directly loading or storing a packed field, the
compiler might be able to paper over alignment issues with shifts and masks.
However if you take a reference to a packed field, it's unlikely that the
compiler will be able to emit code to avoid an unaligned load.

**[As of Rust 1.0 this can cause undefined behavior.][ub loads]**

`repr(packed)` is not to be used lightly. Unless you have extreme requirements,
this should not be used.

This repr is a modifier on `repr(C)` and `repr(rust)`.

[drop flags]: drop-flags.html
[ub loads]: https://github.com/rust-lang/rust/issues/27060
