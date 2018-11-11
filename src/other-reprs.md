# Alternative representations

Rust allows you to specify alternative data layout strategies from the default.
There's also the [reference].




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

* DST pointers (fat pointers), tuples, and enums with fields are not a concept
  in C, and as such are never FFI-safe.

* If `T` is an [FFI-safe non-nullable pointer
  type](ffi.html#the-nullable-pointer-optimization),
  `Option<T>` is guaranteed to have the same layout and ABI as `T` and is
  therefore also FFI-safe. As of this writing, this covers `&`, `&mut`,
  and function pointers, all of which can never be null.

* Tuple structs are like structs with regards to `repr(C)`, as the only
  difference from a struct is that the fields aren’t named.

* This is equivalent to one of `repr(u*)` (see the next section) for enums. The
chosen size is the default enum size for the target platform's C application
binary interface (ABI). Note that enum representation in C is implementation
defined, so this is really a "best guess". In particular, this may be incorrect
when the C code of interest is compiled with certain flags.

* Field-less enums with `repr(C)` or `repr(u*)` still may not be set to an
integer value without a corresponding variant, even though this is
permitted behavior in C or C++. It is undefined behavior to (unsafely)
construct an instance of an enum that does not match one of its
variants. (This allows exhaustive matches to continue to be written and
compiled as normal.)



# repr(transparent)

This can only be used on structs with a single non-zero-sized field (there may
be additional zero-sized fields). The effect is that the layout and ABI of the
whole struct is guaranteed to be the same as that one field.

The goal is to make it possible to transmute between the single field and the
struct. An example of that is the [`UnsafeCell`], which can be transmuted into
the type it wraps.

Also, passing the struct through FFI where the inner field type is expected on
the other side is allowed. In particular, this is necessary for `struct
Foo(f32)` to have the same ABI as `f32`.

More details are in the [RFC][rfc-transparent].



# repr(u*), repr(i*)

These specify the size to make a field-less enum. If the discriminant overflows
the integer it has to fit in, it will produce a compile-time error. You can
manually ask Rust to allow this by setting the overflowing element to explicitly
be 0. However Rust will not allow you to create an enum where two variants have
the same discriminant.

The term "field-less enum" only means that the enum doesn't have data in any
of its variants. A field-less enum without a `repr(u*)` or `repr(C)` is
still a Rust native type, and does not have a stable ABI representation.
Adding a `repr` causes it to be treated exactly like the specified
integer size for ABI purposes.

Any enum with fields is a Rust type with no guaranteed ABI (even if the
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

**[As of Rust 1.30.0 this still can cause undefined behavior.][ub loads]**

`repr(packed)` is not to be used lightly. Unless you have extreme requirements,
this should not be used.

This repr is a modifier on `repr(C)` and `repr(rust)`.

# repr(align(n))

`repr(align(n))` (where `n` is a power of two) forces the type to have an
alignment of *at least* n.

This enables several tricks, like making sure neighboring elements of an array
never share the same cache line with each other (which may speed up certain
kinds of concurrent code).

This is a modifier on `repr(C)` and `repr(rust)`. It is incompatible with
`repr(packed)`.

[reference]: https://github.com/rust-rfcs/unsafe-code-guidelines/tree/master/reference/src/representation
[drop flags]: drop-flags.html
[ub loads]: https://github.com/rust-lang/rust/issues/27060
[`UnsafeCell`]: ../std/cell/struct.UnsafeCell.html
[rfc-transparent]: https://github.com/rust-lang/rfcs/blob/master/text/1758-repr-transparent.md
