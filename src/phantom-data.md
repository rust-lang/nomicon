# PhantomData

불안전한 코드와 작업을 하다 보면, 우리는 종종 타입이나 수명이 구조체와 논리적으로 연관되어 있지만, 실제로 그 필드의 일부분은 아닌 상황을 마주할 수 있습니다. 이런 상황은 보통 수명인 경우가 많은데요, 예를 들어 `&'a [T]`를 위한 `Iter`는 (거의) 다음과 같이 정의되어 있습니다:

```rust,compile_fail
struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
}
```

하지만 `'a`가 구조체의 본문에 쓰이지 않았기 때문에, 이 수명은 *무제한이* 됩니다. [이것이 역사적으로 초래해왔던 문제들 때문에][unused-param], 무제한 수명과 이를 사용하는 타입은 구조체 선언에서 *금지되었습니다*. 따라서 우리는 어떻게든 이 타입들을 구조체 안에서 참조해야 합니다. 
이것을 올바르게 하는 것은 올바른 변성과 해제 검사에 있어서 필수적입니다.

[unused-param]: https://rust-lang.github.io/rfcs/0738-variance.html#the-corner-case-unused-parameters-and-parameters-that-are-only-used-unsafely

우리는 이것을 특별한 표시 타입인 `PhantomData`를 통해서 합니다. `PhantomData`는 공간을 차지하지 않지만, 컴파일러의 분석을 위해 주어진 타입의 필드를 흉내냅니다. 이 방식은 우리가 원하는 변성을 직접 타입 시스템에 말하는 것보다 더 오류에 견고하다고 평가되었습니다. 또한 이 방식은 자동 트레잇과 해제 검사에 필요한 정보 등의 유용한 것들을 컴파일러에게 제공합니다.

`Iter`는 논리적으로 여러 개의 `&'a T`를 포함하므로, 바로 이렇게 우리는 `PhantomData`에게 흉내내라고 할 것입니다:

```rust
use std::marker;

struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
    _marker: marker::PhantomData<&'a T>,
}
```

이렇게만 하면 됩니다. 수명은 제한될 것이고, 반복자는 `'a`와 `T`에 대해서 공변할 것입니다. 모든 게 그냥 마법처럼 동작할 겁니다.

## 제네릭 매개변수와 해제 검사

[RFC 1238](https://rust-lang.github.io/rfcs/1238-nonparametric-dropck.html)의 도움으로, 우리가 이런 코드를 쓴다면:

```rust
struct Vec<T> {
    data: *const T, // 변성을 위한 `*const`
    len: usize,
    cap: usize,
}

#[cfg(any())]
impl<T> Drop for Vec<T> { /* … */ }
```
이 `impl<T> Drop for Vec<T>`의 존재가 러스트로 하여금 `Vec<T>`가 `T` 타입의 값들을 *소유한다고* (더 정확히는: `Drop` 구현에서 `T` 타입의 값들을 사용할 수 있다고) 간주하게 만들고, 따라서 러스트는 `Vec<T>`가 해제될 때 
`T` 타입의 값들이 *달랑거리는* 것을 허용하지 않을 것입니다.

따라서 어떤 타입이 `Drop impl`을 가지고 있다면, **추가적으로 `_owns_T: PhantomData<T>` 필드를 선언하는 것은 *불필요하고* 아무것도 달성하지 않습니다**, 해제 검사기가 볼 때에는요 (변성과 자동 트레잇들에서는 영향을 줍니다).
 
> 특수한 경우: 만약 `PhantomData`를 포함하는 타입이 그 자체로는 `Drop` 구현이 전혀 없지만, `Drop` 구현이 있는 _다른_ 필드를 포함한다면, 여기에 명시된
해제 검사기/`#[may_dangle]` 사항들이 적용될 것입니다: 포함하는 타입이 범위 밖으로 벗어날 때, `PhantomData<T>`
필드는 `T` 타입이 해제되어도 괜찮도록 할 것입니다.
___

하지만 이런 상황은 때때로 과도하게 제한된 코드로 이어질 수 있습니다. 바로 그래서 표준 라이브러리는 불안정하고 `unsafe`한 속성을 써서 바로 이 문서에서 경고했던, 구식의 "수동" 해제 검사 방식으로 돌아가는 겁니다: 
`#[may_dangle]` 속성으로요.

### 예외: 표준 라이브러리의 특수한 경우와 불안정한 `#[may_dangle]`

이 섹션은 당신이 자신의 라이브러리 코드만을 작성한다면 넘어가도 됩니다. 하지만 표준 라이브러리가 실제 `Vec` 정의를 가지고 무엇을 하는지 궁금하다면, 건전성을 위해 여전히 `_owns_T: PhantomData<T>`가 필요하다는 것을 
알아차릴 겁니다.

<details><summary>그 이유를 보려면 클릭하세요</summary>

다음의 예제를 생각해 봅시다:

```rust
fn main() {
    let mut v: Vec<&str> = Vec::new();
    let s: String = "Short-lived".into();
    v.push(&s);
    drop(s);
} // <- `v` 는 여기서 해제됩니다
```

정석적으로 `impl<T> Drop for Vec<T> {`를 정의하면, 위의 코드는 [부정됩니다][is-denied].

[is-denied]: https://rust.godbolt.org/z/ans15Kqz3

Indeed, in this case we have a `Vec</* T = */ &'s str>` vector of `'s`-lived references
to `str`ings, but in the case of `let s: String`, it is dropped before the `Vec` is, and
thus `'s` **is expired** by the time the `Vec` is dropped, and the
`impl<'s> Drop for Vec<&'s str> {` is used.

This means that if such `Drop` were to be used, it would be dealing with an _expired_, or
_dangling_ lifetime `'s`. But this is contrary to Rust principles, where by default all
Rust references involved in a function signature are non-dangling and valid to dereference.

Hence why Rust has to conservatively deny this snippet.

And yet, in the case of the real `Vec`, the `Drop` impl does not care about `&'s str`,
_since it has no drop glue of its own_: it only wants to deallocate the backing buffer.

In other words, it would be nice if the above snippet was somehow accepted, by special
casing `Vec`, or by relying on some special property of `Vec`: `Vec` could try to
_promise not to use the `&'s str`s it holds when being dropped_.

This is the kind of `unsafe` promise that can be expressed with `#[may_dangle]`:

```rust ,ignore
unsafe impl<#[may_dangle] 's> Drop for Vec<&'s str> { /* … */ }
```

or, more generally:

```rust ,ignore
unsafe impl<#[may_dangle] T> Drop for Vec<T> { /* … */ }
```

is the `unsafe` way to opt out of this conservative assumption that Rust's drop
checker makes about type parameters of a dropped instance not being allowed to dangle.

And when this is done, such as in the standard library, we need to be careful in the
case where `T` has drop glue of its own. In this instance, imagine replacing the
`&'s str`s with a `struct PrintOnDrop<'s> /* = */ (&'s str);` which would have a
`Drop` impl wherein the inner `&'s str` would be dereferenced and printed to the screen.

Indeed, `Drop for Vec<T> {`, before deallocating the backing buffer, does have to transitively
drop each `T` item when it has drop glue; in the case of `PrintOnDrop<'s>`, it means that
`Drop for Vec<PrintOnDrop<'s>>` has to transitively drop the `PrintOnDrop<'s>`s elements before
deallocating the backing buffer.

So when we said that `'s` `#[may_dangle]`, it was an excessively loose statement. We'd rather want
to say: "`'s` may dangle provided it not be involved in some transitive drop glue". Or, more generally,
"`T` may dangle provided it not be involved in some transitive drop glue". This "exception to the
exception" is a pervasive situation whenever **we own a `T`**. That's why Rust's `#[may_dangle]` is
smart enough to know of this opt-out, and will thus be disabled _when the generic parameter is held
in an owned fashion_ by the fields of the struct.

Hence why the standard library ends up with:

```rust
# #[cfg(any())]
// we pinky-swear not to use `T` when dropping a `Vec`…
unsafe impl<#[may_dangle] T> Drop for Vec<T> {
    fn drop(&mut self) {
        unsafe {
            if mem::needs_drop::<T>() {
                /* … except here, that is, … */
                ptr::drop_in_place::<[T]>(/* … */);
            }
            // …
            dealloc(/* … */)
            // …
        }
    }
}

struct Vec<T> {
    // … except for the fact that a `Vec` owns `T` items and
    // may thus be dropping `T` items on drop!
    _owns_T: core::marker::PhantomData<T>,

    ptr: *const T, // `*const` for variance (but this does not express ownership of a `T` *per se*)
    len: usize,
    cap: usize,
}
```

</details>

___

Raw pointers that own an allocation is such a pervasive pattern that the
standard library made a utility for itself called `Unique<T>` which:

* wraps a `*const T` for variance
* includes a `PhantomData<T>`
* auto-derives `Send`/`Sync` as if T was contained
* marks the pointer as `NonZero` for the null-pointer optimization

## Table of `PhantomData` patterns

Here’s a table of all the wonderful ways `PhantomData` could be used:

| Phantom type                | variance of `'a` | variance of `T`   | `Send`/`Sync`<br/>(or lack thereof)       | dangling `'a` or `T` in drop glue<br/>(_e.g._, `#[may_dangle] Drop`) |
|-----------------------------|:----------------:|:-----------------:|:-----------------------------------------:|:------------------------------------------------:|
| `PhantomData<T>`            | -                | **cov**ariant     | inherited                                 | disallowed ("owns `T`")                          |
| `PhantomData<&'a T>`        | **cov**ariant    | **cov**ariant     | `Send + Sync`<br/>requires<br/>`T : Sync` | allowed                                          |
| `PhantomData<&'a mut T>`    | **cov**ariant    | **inv**ariant     | inherited                                 | allowed                                          |
| `PhantomData<*const T>`     | -                | **cov**ariant     | `!Send + !Sync`                           | allowed                                          |
| `PhantomData<*mut T>`       | -                | **inv**ariant     | `!Send + !Sync`                           | allowed                                          |
| `PhantomData<fn(T)>`        | -                | **contra**variant | `Send + Sync`                             | allowed                                          |
| `PhantomData<fn() -> T>`    | -                | **cov**ariant     | `Send + Sync`                             | allowed                                          |
| `PhantomData<fn(T) -> T>`   | -                | **inv**ariant     | `Send + Sync`                             | allowed                                          |
| `PhantomData<Cell<&'a ()>>` | **inv**ariant    | -                 | `Send + !Sync`                            | allowed                                          |

  - Note: opting out of the `Unpin` auto-trait requires the dedicated [`PhantomPinned`] type instead.

[`PhantomPinned`]: ../core/marker/struct.PhantomPinned.html
