# 다른 데이터 표현들

러스트는 기본으로부터 다른 데이터 설계 전략을 구성하게 해 줍니다. 

[불안전 코드 가이드라인][unsafe_guide]도 있습니다 (**비표준이니** 주의하세요).

## repr(C)

이것을 가장 중요한 `repr`입니다. 이것은 매우 간단한 의도를 가지고 있습니다: C가 하는대로 하라는 것이죠. 필드들의 정렬 순서, 크기, 정렬선은 C나 C++에서 되는 것 같이 될 겁니다. 
어떤 타입이든 FFI 경계를 넘겨 보내려면 `repr(C)`로 표현되어야 하는데, 이는 C가 프로그래밍 세계의 공용어이기 때문입니다. 이것은 값을 다른 타입으로 재해석하는 것과 같은, 데이터 레이아웃을 가지고 정교한 장난을 수월하게 칠 수 있기 위해서 필수적입니다. 

우리는 [rust-bindgen]과 [cbindgen]를 둘 다, 혹은 둘 중 하나를 써서 당신 대신 FFI 경계를 관리하기를 매우 권장합니다. 러스트 팀은 이 프로젝트들과 긴밀하게 작업하여 이들이 튼튼하게 작동하고, 
타입 레이아웃과 `repr`들에 대한 현재와 미래의 보장에 잘 맞도록 신경쓰고 있습니다.

`repr(C)`와 러스트의 (C보다) 이상한 데이터 설계 기능의 상호작용은 주의해야 합니다. "FFI를 위한" 것과 "데이터 표현을 바꾸기" 위한 두 가지 목적이 동시에 있기 때문에, `repr(C)`는 FFI 경계로 보내면 말이 안되거나 문제가 생길 수 있는 타입들에 적용할 수 있습니다.

* 무량 타입(ZST)은 그대로 크기가 0으로 되는데, 이것은 C에서 표준 동작이 아니고, C++에서 빈 타입의 동작과 분명하게 반대되는데, C++에서는 빈 타입이라도 한 바이트의 공간을 차지해야 한다고 말하기 때문입니다.

* 동량 타입(DST)의 포인터(넓은 포인터)와 튜플은 C에서 없는 개념이므로, FFI로 보내면 절대 안전하지 않습니다.

* 필드가 있는 열거형 또한 C와 C++에서 없는 개념이지만, 타입 사이의 유효한 변환이 [정의되어 있습니다][really-tagged].



* If `T` is an [FFI-safe non-nullable pointer
  type](ffi.html#the-nullable-pointer-optimization),
  `Option<T>` is guaranteed to have the same layout and ABI as `T` and is
  therefore also FFI-safe. As of this writing, this covers `&`, `&mut`,
  and function pointers, all of which can never be null.

* Tuple structs are like structs with regards to `repr(C)`, as the only
  difference from a struct is that the fields aren’t named.

* `repr(C)` is equivalent to one of `repr(u*)` (see the next section) for
fieldless enums. The chosen size is the default enum size for the target platform's C
application binary interface (ABI). Note that enum representation in C is implementation
defined, so this is really a "best guess". In particular, this may be incorrect
when the C code of interest is compiled with certain flags.

* Fieldless enums with `repr(C)` or `repr(u*)` still may not be set to an
integer value without a corresponding variant, even though this is
permitted behavior in C or C++. It is undefined behavior to (unsafely)
construct an instance of an enum that does not match one of its
variants. (This allows exhaustive matches to continue to be written and
compiled as normal.)

## repr(transparent)

`#[repr(transparent)]` can only be used on a struct or single-variant enum that has a single non-zero-sized field (there may be additional zero-sized fields).
The effect is that the layout and ABI of the whole struct/enum is guaranteed to be the same as that one field.

> NOTE: There's a `transparent_unions` nightly feature to apply `repr(transparent)` to unions,
> but it hasn't been stabilized due to design concerns. See the [tracking issue][issue-60405] for more details.

The goal is to make it possible to transmute between the single field and the
struct/enum. An example of that is [`UnsafeCell`], which can be transmuted into
the type it wraps ([`UnsafeCell`] also uses the unstable [no_niche][no-niche-pull],
so its ABI is not actually guaranteed to be the same when nested in other types).

Also, passing the struct/enum through FFI where the inner field type is expected on
the other side is guaranteed to work. In particular, this is necessary for
`struct Foo(f32)` or `enum Foo { Bar(f32) }` to always have the same ABI as `f32`.

This repr is only considered part of the public ABI of a type if either the single
field is `pub`, or if its layout is documented in prose. Otherwise, the layout should
not be relied upon by other crates.

More details are in the [RFC 1758][rfc-transparent] and the [RFC 2645][rfc-transparent-unions-enums].

## repr(u*), repr(i*)

These specify the size to make a fieldless enum. If the discriminant overflows
the integer it has to fit in, it will produce a compile-time error. You can
manually ask Rust to allow this by setting the overflowing element to explicitly
be 0. However Rust will not allow you to create an enum where two variants have
the same discriminant.

The term "fieldless enum" only means that the enum doesn't have data in any
of its variants. A fieldless enum without a `repr(u*)` or `repr(C)` is
still a Rust native type, and does not have a stable ABI representation.
Adding a `repr` causes it to be treated exactly like the specified
integer size for ABI purposes.

If the enum has fields, the effect is similar to the effect of `repr(C)`
in that there is a defined layout of the type. This makes it possible to
pass the enum to C code, or access the type's raw representation and directly
manipulate its tag and fields. See [the RFC][really-tagged] for details.

These `repr`s have no effect on a struct.

Adding an explicit `repr(u*)`, `repr(i*)`, or `repr(C)` to an enum with fields suppresses the null-pointer optimization, like:

```rust
# use std::mem::size_of;
enum MyOption<T> {
    Some(T),
    None,
}

#[repr(u8)]
enum MyReprOption<T> {
    Some(T),
    None,
}

assert_eq!(8, size_of::<MyOption<&u16>>());
assert_eq!(16, size_of::<MyReprOption<&u16>>());
```

This optimization still applies to fieldless enums with an explicit `repr(u*)`, `repr(i*)`, or `repr(C)`.

## repr(packed)

`repr(packed)` forces Rust to strip any padding, and only align the type to a
byte. This may improve the memory footprint, but will likely have other negative
side-effects.

In particular, most architectures *strongly* prefer values to be aligned. This
may mean the unaligned loads are penalized (x86), or even fault (some ARM
chips). For simple cases like directly loading or storing a packed field, the
compiler might be able to paper over alignment issues with shifts and masks.
However if you take a reference to a packed field, it's unlikely that the
compiler will be able to emit code to avoid an unaligned load.

[As this can cause undefined behavior][ub loads], the lint has been implemented
and it will become a hard error.

`repr(packed)` is not to be used lightly. Unless you have extreme requirements,
this should not be used.

This repr is a modifier on `repr(C)` and `repr(Rust)`.

## repr(align(n))

`repr(align(n))` (where `n` is a power of two) forces the type to have an
alignment of *at least* n.

This enables several tricks, like making sure neighboring elements of an array
never share the same cache line with each other (which may speed up certain
kinds of concurrent code).

This is a modifier on `repr(C)` and `repr(Rust)`. It is incompatible with
`repr(packed)`.

[unsafe_guide]: https://rust-lang.github.io/unsafe-code-guidelines/layout.html
[drop_flags]: drop-flags.html
[ub_loads]: https://github.com/rust-lang/rust/issues/27060
[issue-60405]: https://github.com/rust-lang/rust/issues/60405
[`UnsafeCell`]: ../std/cell/struct.UnsafeCell.html
[rfc-transparent]: https://github.com/rust-lang/rfcs/blob/master/text/1758-repr-transparent.md
[rfc-transparent-unions-enums]: https://rust-lang.github.io/rfcs/2645-transparent-unions.html
[really-tagged]: https://github.com/rust-lang/rfcs/blob/master/text/2195-really-tagged-unions.md
[rust-bindgen]: https://rust-lang.github.io/rust-bindgen/
[cbindgen]: https://github.com/eqrion/cbindgen
[no-niche-pull]: https://github.com/rust-lang/rust/pull/68491
