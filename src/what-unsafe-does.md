# 불안전한 러스트는 무엇을 할 수 있는가

불안전한 러스트에서 다른 점은 이런 것들이 가능하다는 것뿐입니다:

* 생포인터 역참조하기
* `unsafe` 함수 호출하기 (C 함수나, 컴파일러 내부, 그리고 할당자를 직접 호출하는 것 포함)
* `unsafe` 트레잇 구현하기
* `static` 변수의 값을 변경하기
* `union` 의 필드를 접근하기

이게 답니다. 이런 연산들이 불안전의 영역으로 추방된 이유는, 이것들 중 하나라도 잘못 사용할 경우 그토록 두렵던 미정의 동작을 일으키기 때문입니다. 
미정의 동작을 일으키면 컴파일러가 당신의 프로그램에 임의의 나쁜 짓들을 할 수 있는 모든 권리를 얻게 됩니다. 당연하게도 미정의 동작은 *일으켜서는 안됩니다.*

C와 다르게, 미정의 동작은 러스트에서는 꽤 제한되어 있습니다. 러스트의 코어 언어가 막으려고 하는 것들은 이런 것들입니다: 

* 달랑거리거나 정렬되어 있지 않은 포인터를 역참조하는 것 (`*` 연산자를 사용해서) (밑 참조)
* [레퍼런스 규칙][alias] 을 지키지 않는 것
* 잘못된 호출 ABI를 이용해 함수를 호출하거나 잘못된 되감기 ABI를 가지고 있는 함수에서 되감는 것
* [데이터 경합][race] 을 일으키는 것
* 지금 실행하는 스레드가 지원하지 않는 [타겟 기능들][target] 로 컴파일된 코드를 실행하는 것

* Producing invalid values (either alone or as a field of a compound type such
  as `enum`/`struct`/array/tuple):
  * a `bool` that isn't 0 or 1
  * an `enum` with an invalid discriminant
  * a null `fn` pointer
  * a `char` outside the ranges [0x0, 0xD7FF] and [0xE000, 0x10FFFF]
  * a `!` (all values are invalid for this type)
  * an integer (`i*`/`u*`), floating point value (`f*`), or raw pointer read from
    [uninitialized memory][], or uninitialized memory in a `str`.
  * a reference/`Box` that is dangling, unaligned, or points to an invalid value.
  * a wide reference, `Box`, or raw pointer that has invalid metadata:
    * `dyn Trait` metadata is invalid if it is not a pointer to a vtable for
      `Trait` that matches the actual dynamic trait the pointer or reference points to
    * slice metadata is invalid if the length is not a valid `usize`
      (i.e., it must not be read from uninitialized memory)
  * a type with custom invalid values that is one of those values, such as a
    [`NonNull`] that is null. (Requesting custom invalid values is an unstable
    feature, but some stable libstd types, like `NonNull`, make use of it.)

For a more detailed explanation about "Undefined Bahavior", you may refer to
[the reference][behavior-considered-undefined].

"Producing" a value happens any time a value is assigned, passed to a
function/primitive operation or returned from a function/primitive operation.

A reference/pointer is "dangling" if it is null or not all of the bytes it
points to are part of the same allocation (so in particular they all have to be
part of *some* allocation). The span of bytes it points to is determined by the
pointer value and the size of the pointee type. As a consequence, if the span is
empty, "dangling" is the same as "null". Note that slices and strings point
to their entire range, so it's important that the length metadata is never too
large (in particular, allocations and therefore slices and strings cannot be
bigger than `isize::MAX` bytes). If for some reason this is too cumbersome,
consider using raw pointers.

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
* Overflow integers (with the built-in operators such as `+` etc.)
* Abort the program
* Delete the production database

For more detailed information, you may refer to [the reference][behavior-not-considered-unsafe].

However any program that actually manages to do such a thing is *probably*
incorrect. Rust provides lots of tools to make these things rare, but
these problems are considered impractical to categorically prevent.

[alias]: references.html
[uninit]: uninitialized.html
[race]: races.html
[target]: ../reference/attributes/codegen.html#the-target_feature-attribute
[`NonNull`]: ../std/ptr/struct.NonNull.html
[behavior-considered-undefined]: ../reference/behavior-considered-undefined.html
[behavior-not-considered-unsafe]: ../reference/behavior-not-considered-unsafe.html
