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

확실히 이런 경우에서는, 우리는 `Vec<&'s str>`, 즉 `str`의 `'s`만큼 사는 레퍼런스들의 벡터를 가지고 있습니다. 하지만 `let s: String`에서는, 이것이 `Vec`보다 먼저 해제되고, 
`impl<'s> Drop for Vec<&'s str> {`의 정의가 사용됩니다.

이것이 의미하는 것은 만약 이런 `Drop` 구현이 사용된다면, _파기된_, 혹은 _달랑거리는_ 수명 `'s`로 작업을 할 것이라는 점입니다. 하지만 이것은 함수 시그니처에 있는 모든 러스트 레퍼런스는 기본적으로 달랑거리지 않고 역참조해도 문제가 없다는 
러스트 규칙에 반대됩니다.

따라서 러스트는 보수적으로 이 코드를 부정할 수밖에 없습니다.

그런데 실제 `Vec`의 경우에서, `Drop` 구현은 `&'s str`에 대해 신경쓰지 않는데, _이는 `&'s str`이 따로 `Drop` 구현이 없기 때문입니다_: `Vec`의 `Drop` 구현은 그저 버퍼를 해제하고 싶을 뿐이죠.

즉, `Vec`의 경우를 특별하게 구분해서, 또는 `Vec`의 특수한 성질을 이용해서 위의 코드가 컴파일되면 좋겠네요: `Vec`이 _가지고 있는 `&'s str`들을 해제될 때 사용하지 않도록 약속할 수도 있겠어요_.

이 약속은 `#[may_dangle]`로 표현될 수 있는 `unsafe`한 약속입니다:

```rust ,ignore
unsafe impl<#[may_dangle] 's> Drop for Vec<&'s str> { /* … */ }
```

아니면 좀더 일반적으로 표현하자면:

```rust ,ignore
unsafe impl<#[may_dangle] T> Drop for Vec<T> { /* … */ }
```

이것이 러스트의 해제 검사기가 해제되는 값의 타입 매개변수가 달랑거리지 않도록 하는, 보수적인 추측에서 탈출하도록 하는 `unsafe`한 방법입니다.

표준 라이브러리와 같이 이렇게 했다면, 우리는 `T`가 자체의 `Drop` 구현이 있는 경우를 조심해야 합니다. 이 경우에는, `&'s str`를 `struct PrintOnDrop<'s>(&'s str);`로 바꾸는 것을 상상해 봅시다. 
이 구조체는 자체의 `Drop` 구현에서 내부의 `&'s str`를 역참조하여 화면에 출력할 것입니다.

확실히 버퍼를 해제하기 전에 `Drop for Vec<T> {`는, 내부의 각 `T`들이 `Drop` 구현이 있을 때, 각 `T`들을 해제시켜야 합니다. `PrintOnDrop<'s>`의 경우에 `Drop for Vec<PrintOnDrop<'s>>`는 
버퍼를 해제하기 전에 `PrintOnDrop<'s>`의 원소들을 해제시켜야 합니다.

따라서 우리가 `'s`가 `#[may_dangle]`하다고 말할 때, 이것은 심하게 모호하게 말했던 것입니다. 우리는 대신 이렇게 말해야 할 것입니다: "`'s`는 `Drop` 구현에 구속받지 않는 한에서 달랑거릴 수도 있습니다". 
혹은 더 일반적으로 이렇게요: "`T`는 `Drop` 구현에 구속받지 않는 한에서 달랑거릴 수도 있습니다". 이런 "예외의 예외"는 **우리가 `T`를 소유할 때마다** 발생하는 흔한 현상입니다. 이래서 러스트의 `#[may_dangle]`은 
이런 예외 상황에 대해 알고, 따라서 구조체의 필드들에 _제네릭 매개변수가 소유되는 때에_ 비활성화될 것입니다.

따라서 표준 라이브러리는 이렇게 결론을 내립니다:

```rust
#[cfg(any())]
// 우리는 `Vec`을 해제할 때 `T`를 사용하지 않도록 약속합니다…
unsafe impl<#[may_dangle] T> Drop for Vec<T> {
    fn drop(&mut self) {
        unsafe {
            if mem::needs_drop::<T>() {
                /* … 이 경우는 제외하고요, 어떤 경우냐면 … */
                ptr::drop_in_place::<[T]>(/* … */);
            }
            // …
            dealloc(/* … */)
            // …
        }
    }
}

struct Vec<T> {
    // … `Vec`이 `T`의 원소들을 소유하고,
    // 따라서 해제될 때 `T`의 원소들을 해제시킬 때 말이죠!
    _owns_T: core::marker::PhantomData<T>,

    ptr: *const T, // 변성을 위한 `*const`입니다 (하지만 이것이 *그 자체로* `T`의 소유권을 나타내는 것은 아닙니다)
    len: usize,
    cap: usize,
}
```

</details>

___

할당된 메모리를 소유하는 생 포인터는 너무나 흔한 패턴이여서, 표준 라이브러리는 이것을 위한 도구인 `Unique<T>`를 만들었는데, 이것은:

* 변성을 위해 `*const T`를 안에 포함합니다
* `PhantomData<T>`를 포함합니다
* 마치 `T`가 포함된 것처럼 `Send`/`Sync`를 자동으로 구현합니다
* 널 포인터 최적화를 위해 포인터를 `NonZero`로 표시합니다

## `PhantomData` 패턴의 표

여기 `PhantomData`가 사용될 수 있는 모든 경우의 놀라운 표가 있습니다:

| 흉내내는 타입                   | `'a`의 변성       | `T`의 변성          | `Send`/`Sync`<br/>(or lack thereof)       | dangling `'a` or `T` in drop glue<br/>(_e.g._, `#[may_dangle] Drop`) |
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
