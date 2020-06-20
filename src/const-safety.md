# Const safety

The miri engine, which is used to execute code at compile time, can fail in
four possible ways:

* The program performs an unsupported operation (e.g., calling an unimplemented
  intrinsics, or doing an operation that would observe the integer address of a
  pointer).
* The program causes undefined behavior (e.g., dereferencing an out-of-bounds
  pointer).
* The program panics (e.g., a failed bounds check).
* The program exhausts its resources: It might overflow the stack, allocation
  too much memory or loops forever.  Note that detecting these conditions
  happens on a best-effort basis only.

Just like panics and non-termination are acceptable in safe run-time Rust code,
we also consider these acceptable in safe compile-time Rust code.  However, we
would like to rule out the first two kinds of failures in safe code.  Following
the terminology in [this blog post], we call a program that does not fail in the
first two ways *const safe*.

[this blog post]: https://www.ralfj.de/blog/2018/07/19/const.html

The goal of the const safety check, then, is to ensure that a program is const
safe.  What makes this tricky is that there are some operations that are safe as
far as run-time Rust is concerned, but are inherently unsupportable in the miri engine and hence
not const safe (they fall in the first category of failures above).
We call these operations *unconst*.  The purpose
of the following section is to explain this in more detail, before proceeding
with the main definitions.

## Miri background

A very simple example of an unconst operation is

```rust
static S: i32 = 0;
const BAD: bool = (&S as *const i32 as usize) % 16 == 0;
```

The modulo operation here is not supported by the miri engine because evaluating
it requires knowing the actual integer address of `S`.

The way miri handles this is by treating pointer and integer values separately.
The most primitive kind of value in miri is a `Scalar`, and a scalar is *either*
a pointer (`Scalar::Ptr`) *or* a bunch of bits representing an integer
(`Scalar::Bits`).  Every value of a variable of primitive type is stored as a
`Scalar`.  In the code above, casting the pointer `&S` to `*const i32` and then
to `usize` does not actually change the value -- we end up with a local variable
of type `usize` whose value is a `Scalar::Ptr`.  This is not a problem in
itself, but then executing `%` on this *pointer value* is unsupported.

However, it does not seem appropriate to blame the `%` operation above for this
failure. `%` on "normal" `usize` values (`Scalar::Bits`) is perfectly fine, just using it on
values computed from pointers is an issue.  Essentially, `&i32 as *const i32 as
usize` is a "safe" `usize` at run-time (meaning that applying safe operations to
this `usize` cannot lead to misbehavior, following terminology [suggested here])
-- but the same value is *not* "safe" at compile-time, because we can cause a
const safety violation by applying a safe operation (namely, `%`).

This is similar to how it is illegal in Rust to have a dangling reference, because you
can safely dereference it, which would be undefined behavior. Even though the issue
occurs at the time of the dereference, we treat the creation of a dangling reference
as the unsound action, not the final dereference.

[suggested here]: https://www.ralfj.de/blog/2018/08/22/two-kinds-of-invariants.html

## Const safety check on values

The result of any const computation (`const`, `static`, promoteds) is subject to
a "sanity check" which enforces const safety.  Const safety
is defined as follows:

* Integer and floating point types are const-safe if they are a `Scalar::Bits`.
  This makes sure that we can run `%` and other operations without violating
  const safety.  In particular, the value must *not* be uninitialized.
* References are const-safe if they are `Scalar::Ptr` into allocated memory, and
  the data stored there is const-safe.  (Technically, we would also like to
  require `&mut` to be unique and the memory behind a `&` to not be mutable unless there is an
  `UnsafeCell`, but it seems infeasible to check that.)  For wide pointers, the
  length of a slice must be a valid `usize` and the vtable of a `dyn Trait` must
  be a valid vtable.
* `bool` is const-safe if it is `Scalar::Bits` with a value of `0` or `1`.
* `char` is const-safe if it is a valid unicode codepoint.
* `()` is always const-safe.
* `!` is never const-safe.
* `dyn Trait` is const-safe if the value is const-safe at the type indicated by
  the vtable.
* Function pointers are const-safe if they point to an actual function.  A
  `const fn` pointer (when/if we have those) must point to a `const fn`.
* Raw pointers are const safe if they are either `Scalar::Bits` or are not dangling
* Aggregates must not contain dangling `Scalar::Ptr` (e.g. in padding or unions) and additionally
  follow the following aggregate-specific rules:
    * Tuples, structs, arrays and slices are const-safe if all their fields are
    const-safe.
    * Enums are const-safe if they have a valid discriminant and the fields of the
    active variant are const-safe.
    * Unions are always const-safe; the data does not matter.


For example:

```rust
static S: i32 = 0;
const BAD: usize = unsafe { &S as *const i32 as usize };
```

Here, `S` is const-safe because `0` is a `Scalar::Bits`.  However, `BAD` is *not* const-safe because it is a `Scalar::Ptr`.
Also in

```rust
static X: i32 = 42;
static GOOD: *const i32 = &X;
static BAD: *const i32 = { let x = 42; &x as *const i32 };
```

`BAD` is not const-safe, because it points to the local variable `x` which will be deallocated, while
`GOOD` is const-safe, because it points to some memory that lives as long as the static itself.


## Const safety check on code

The purpose of the const safety check on code is to prohibit construction of
non-const-safe values in safe code.  We can allow *almost* all runtime-safe operations,
except for unconst operations -- which are all related to raw pointers:

* Comparing raw pointers for (in)equality or order,
* converting raw pointers to integers,
* inspecting the bits of raw pointers (e.g. by hashing them).


These operations are unconst because a `const fn` that
(when called with the same arguments at runtime and compile-time)

  * fails to run at compile time when it works perfectly fine at runtime
  * produces a different result at compile time than at runtime

is undefined behavior.

`unsafe` blocks permit performing possibly unconst operations.
At this point, it becomes the responsibility of the
programmer to preserve const safety.  In particular, a *safe* `const fn` must
always execute const-safely when called with const-safe arguments, and produce a
const-safe result.  For example, the following function is const-safe (after
some extensions of the miri engine that are already implemented in miri) even
though it uses raw pointer operations:

```rust
const fn slice_eq(x: &[u32], y: &[u32]) -> bool {
    if x.len() != y.len() {
        return false;
    }
    // equal length and address -> memory must be equal, too
    if unconst { x as *const [u32] as *const u32 == y as *const [u32] as *const u32 } {
        return true;
    }
    // assume the following is legal const code for the purpose of this function
    x.iter().eq(y.iter())
}
```

On the other hand, the following function is *not* const-safe and hence it is considered a bug to mark it as such:

```
const fn ptr_eq<T>(x: &T, y: &T) -> bool {
    unconst { x as *const T == y as *const T }
}
```

If the function were invoked as `ptr_eq(&42, &42)` the result depends on the potential
deduplication of the memory of the `42`s.
