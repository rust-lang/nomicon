# 다른 데이터 표현들

러스트는 기본으로부터 다른 데이터 설계 전략을 구성하게 해 줍니다. 

[불안전 코드 가이드라인][unsafe_guide]도 있습니다 (**비표준이니** 주의하세요).

## repr(C)

이것은 가장 중요한 `repr`입니다. 이것은 매우 간단한 의도를 가지고 있습니다: C가 하는대로 하라는 것이죠. 필드들의 정렬 순서, 크기, 정렬선은 C나 C++에서 되는 것 같이 될 겁니다. 
어떤 타입이든 FFI 경계를 넘겨 보내려면 `repr(C)`로 표현되어야 하는데, 이는 C가 프로그래밍 세계의 공용어이기 때문입니다. 이것은 값을 다른 타입으로 재해석하는 것과 같은, 데이터 레이아웃을 가지고 정교한 장난을 수월하게 칠 수 있기 위해서 필수적입니다. 

우리는 [rust-bindgen]과 [cbindgen]를 둘 다, 혹은 둘 중 하나를 써서 당신 대신 FFI 경계를 관리하기를 매우 권장합니다. 러스트 팀은 이 프로젝트들과 긴밀하게 작업하여 이들이 튼튼하게 작동하고, 
타입 레이아웃과 `repr`들에 대한 현재와 미래의 보장에 잘 맞도록 신경쓰고 있습니다.

`repr(C)`와 러스트의 (C보다) 이상한 데이터 설계 기능의 상호작용은 주의해야 합니다. "FFI를 위한" 것과 "데이터 표현을 바꾸기" 위한 두 가지 목적이 동시에 있기 때문에, `repr(C)`는 FFI 경계로 보내면 말이 안되거나 문제가 생길 수 있는 타입들에 적용할 수 있습니다.

* 무량 타입(ZST)은 그대로 크기가 0으로 되는데, 이것은 C에서 표준 동작이 아니고, C++에서 빈 타입의 동작과 분명하게 반대되는데, C++에서는 빈 타입이라도 한 바이트의 공간을 차지해야 한다고 말하기 때문입니다.

* 동량 타입(DST)의 포인터(넓은 포인터)와 튜플은 C에서 없는 개념이므로, FFI로 보내면 절대 안전하지 않습니다.

* 필드가 있는 열거형 또한 C와 C++에서 없는 개념이지만, 타입 사이의 유효한 변환이 [정의되어 있습니다][really-tagged].

* 만약 `T`가 [FFI로 보내도 안전하고 널이 아닌 포인터 타입이라면](ffi.html#the-nullable-pointer-optimization), `Option<T>`는 `T`와 같은 데이터 표현과 ABI를 갖추고, 따라서 FFI로 보내도 안전하다는 것이 보장됩니다. 이 글을 쓰는 시점에서, 이것은 `&`, `&mut`, 그리고 함수 포인터들에 해당하는데, 이것들은 전부 널이 될 수 없기 때문입니다.

* 튜플 구조체는 `repr(C)`에서는 일반 구조체와 같은데, 일반 구조체와 다른 점은 필드의 이름이 없다는 것뿐이기 때문입니다.

* 필드가 없는 열거형에 있어서는 `repr(C)`는 `repr(u*)` (다음 섹션을 보세요) 중 하나와 같습니다. 여기서 선택되는 바이트 크기는 타겟 플랫폼의 C 애플리케이션 이진 인터페이스(ABI)에서의 기본 열거형 크기입니다. 주의할 점은 C에서의 데이터 표현은 구현에 따라 다르게 정의되어 있으므로, 이것은 "최선의 추측"이라는 점입니다. 특별히, 관련된 C 코드가 특정한 플래그로 컴파일되면 이 설명이 맞지 않을 수도 있습니다.

* `repr(C)`나 `repr(u*)`로 표현되는 필드 없는 열거형은, C나 C++에서는 허용되는 동작이지만, 그래도 대응하는 형이 없는 정수 값으로 설정하면 안됩니다. 열거형의 형이 대응하지 않는 열거형의 값을 (불안전하게) 만들어내는 것은 미정의 동작입니다. (이렇게 함으로써 패턴 완전 매칭이 잘 작성되고 컴파일되게 됩니다.)

## repr(transparent)

`#[repr(transparent)]`은 크기가 0이 아닌 하나의 필드를 가지고 있는 (무량 필드는 더 있어도 됩니다) 구조체나 형이 하나인 열거형에만 쓰일 수 있습니다. 
이것의 효과는 구조체/열거형의 전체 레이아웃과 ABI가 그 하나의 필드와 완전히 동일하도록 보장된다는 것입니다.

> 주의: `repr(transparent)`를 공용체에 적용하는 `transparent_unions` nightly 기능이 있지만,
> 디자인 문제 때문에 표준화되지 않았습니다. 더 자세한 내용은 [이슈][issue-60405]를 참고하세요.

이것의 목표는 구조체/열거형과 그의 유일한 필드 간에 변환을 가능하게 하는 것입니다. 이것의 한 예는 [`UnsafeCell`]인데, 이것은 자신이 감싸고 있는 타입으로 변환될 수 있습니다 ([`UnsafeCell`]은 또한 불안정한 [no_niche][no-niche-pull]를 쓰기 때문에, 이것의 ABI는 다른 타입 속에 들어갔을 때에 동일하다고 보장되지는 않습니다).

또, 그 유일한 필드가 FFI로 보내도 괜찮은 타입이라면, 그 유일한 필드가 있는 구조체/열거형을 FFI로 보내는 작업도 잘 작동하는 것이 보장됩니다. 예를 들어, 이것은 `struct Foo(f32)`나 `enum Foo { Bar(f32) }`가 `f32`와 항상 동일한 ABI를 가지게 하도록 하기 위해서 꼭 필요합니다.



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

[As this can cause undefined behavior][ub_loads], the lint has been implemented
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
[`UnsafeCell`]: https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html
[rfc-transparent]: https://github.com/rust-lang/rfcs/blob/master/text/1758-repr-transparent.md
[rfc-transparent-unions-enums]: https://rust-lang.github.io/rfcs/2645-transparent-unions.html
[really-tagged]: https://github.com/rust-lang/rfcs/blob/master/text/2195-really-tagged-unions.md
[rust-bindgen]: https://rust-lang.github.io/rust-bindgen/
[cbindgen]: https://github.com/eqrion/cbindgen
[no-niche-pull]: https://github.com/rust-lang/rust/pull/68491
