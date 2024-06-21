# 안전함과 불안전함은 어떻게 상호작용하는가

안전한 러스트와 불안전한 러스트는 어떤 관계일까요? 둘은 어떻게 상호작용할까요?

안전한 러스트와 불안전한 러스트 간의 구분은 `unsafe` 라는 키워드로 제어되는데, 이것은 서로에게 인터페이스 역할을 합니다. 
이것이 바로 안전한 러스트는 안전한 언어라고 할 수 있는 이유입니다: 모든 불안전한 부분은 `unsafe` 라는 경계 뒤로 밀리거든요. 
원한다면 `#![forbid(unsafe_code)]` 를 코드베이스에 집어넣음으로써 오직 안전한 러스트만 쓴다는 것을 컴파일할 때 보장할 수도 있죠.

`unsafe` 키워드는 두 가지 용도가 있습니다: 컴파일러가 확인할 수 없는 계약의 존재를 정의할 때 사용하고, 또한 
이 계약들이 성립한다는 것을 프로그래머가 확인했다고 선언할 때 사용합니다.

_함수들_ 과 _트레잇 정의들_ 에서 확인되지 않은 계약들의 존재를 알리기 위해 `unsafe` 키워드를 쓸 수 있습니다. 
함수에서 `unsafe` 는 함수의 사용자들이 함수의 문서를 확인해서, 함수가 요구하는 계약을 지키는 방식으로 사용해야 
한다는 것을 의미합니다. 트레잇 정의에서 `unsafe` 는 트레잇의 구현자들이 트레잇 문서를 확인해서 그들의 구현이 
트레잇이 요구하는 계약을 지키는 것을 확실히 해야 한다는 것을 뜻합니다.

코드 블럭에도 `unsafe` 를 사용해서 그 안에서 이루어진 모든 불안전한 작업들이 그 작업들의 계약들을 지켰다는 
것을 확인했다고 선언할 수 있습니다. 예를 들어, [`slice::get_unchecked`][get_unchecked] 에 넘겨진 인덱스는 
범위 안에 있어야 합니다.

트레잇 구현에 `unsafe` 를 사용해서 그 구현이 트레잇의 계약을 지킨다고 선언할 수 있습니다. 예를 들어, [`Send`] 를 
구현하는 타입은 정말로 다른 스레드로 안전하게 이동할 수 있어야 합니다.

표준 라이브러리는 다음을 포함한 다수의 불안전한 함수들을 가지고 있습니다:

* [`slice::get_unchecked`][get_unchecked] 는 범위를 확인하지 않고 인덱싱을 하기 때문에 메모리 안정성이 자유롭게 침해되도록 허용합니다.
* [`mem::transmute`][transmute] 는 어떤 값을 주어진 타입으로 재해석하여 임의의 방식으로 타입 안정성을 건너뜁니다 (자세한 사항은 [변환][conversions] 을 참고하세요).
* 사이즈가 정해진 타입의 모든 생(raw)포인터는 [`offset`][ptr_offset] 메서드가 있는데, 이 메서드는 전달된 편차(offset)가 ["범위 안에 있지"][ptr_offset] 않을 경우 미정의 동작을 일으킵니다.
* 모든 외부 함수 인터페이스 (FFI) 함수들은 호출하기에 `불안전` 합니다. 이는 다른 언어들이 러스트 컴파일러가 확인할 수 없는 임의의 연산들을 할 수 있기 때문입니다.

러스트 1.48.0 버전에서 표준 라이브러리는 다음의 불안전한 트레잇들을 정의하고 있습니다 (다른 것들도 있지만 아직 안정화되지 않았고, 어떤 것들은 나중에도 안정화되지 않을 것입니다):

* [`Send`] 는 이를 구현하는 타입들이 다른 스레드로 이동해도 안전함을 약속하는 표시 트레잇(API가 없는 트레잇)입니다.
* [`Sync`] 는 또다른 표시 트레잇으로, 이를 구현하는 타입들을 불변 레퍼런스를 이용해 스레드들이 서로 공유할 수 있음을 약속합니다.
* [`GlobalAlloc`] 은 프로그램 전체의 메모리 할당자를 커스터마이징할 수 있게 해 줍니다.
* [`SliceIndex`] 는 슬라이스 타입들의 인덱싱을 위한 동작을 정의합니다. 여기에는 경계를 확인하지 않고 인덱싱하는 작업이 포함됩니다.

러스트 표준 라이브러리도 내부적으로 불안전한 러스트를 꽤 많이 씁니다. 이 구현사항들은 수동으로 엄격하게 확인되어서, 
이 위에 안전한 러스트로 지은 인터페이스들은 안전하다고 생각해도 됩니다.

이런 구분의 필요성은 *견고성* 이라고 불리는, 안전한 러스트의 근본적인 특성으로 귀결됩니다: 

**무슨 일을 하던, 안전한 러스트는 미정의 동작을 유발할 수 없습니다.**

안전/불안전으로 구분하는 디자인은 안전한 러스트와 불안전한 러스트 사이에 비대칭적 신뢰 관계가 있다는 것을 의미합니다. 
안전한 러스트는 본질적으로 모든 불안전한 러스트 코드가 올바르게 작성되었다고 믿어야 합니다. 
반면 불안전한 러스트는 부주의하게 작성한 안전한 러스트 코드를 믿을 수 없습니다.

예를 들어, 러스트는 "그냥" 비교할 수 있는 타입과 "완전한" 순서를 가지고 있는 (즉 비교가 합리적으로 이루어지는) 
타입을 구분하기 위해 [`PartialOrd`] 와 [`Ord`] 트레잇을 가지고 있습니다.

[`BTreeMap`] 은 불완전한 순서를 가지는 타입들에 쓰는 것은 말이 안 되기 때문에 키 역할을 하는 타입이 `Ord` 를 구현하도록 요구합니다. 
하지만 `BTreeMap` 은 구현 내부에 불안전한 러스트 코드가 있습니다. 안전한 러스트 코드이긴 하겠지만, 부주의한 `Ord` 구현이 미정의 동작을 일으키는 것은 받아들일 수 없기 때문에, 
`BTreeMap` 에 있는 불안전한 코드는 완전하게 순서를 이루고 있지 않은 `Ord` 구현을 견딜 수 있도록 작성되어야 합니다 - 비록 그렇기 때문에 `Ord` 를 요구한다고 해도요.

불안전한 러스트 코드는 안전한 러스트 코드가 잘 작성되었을 것이라고 마냥 믿을 수가 없습니다. 그래서 말하자면, `BTreeMap` 은 당신이 완전한 순서를 이루지 않는 값들을 집어넣으면 완전히 예측 불가능하게 행동할 겁니다. 다만 미정의 동작은 절대로 일으키지 않을 겁니다.

이렇게 질문할 수도 있습니다, 만약 `BTreeMap` 이 `Ord` 가 안전해서 믿을 수 없다면, *다른* 안전한 코드는 어떻게 믿죠? 예를 들어 `BTreeMap` 은 정수들과 슬라이스 타입들이 올바르게 구현되었을 거라고 가정합니다. 그것들도 안전하잖아요, 그죠?

그 차이는 범위의 차이입니다. `BTreeMap` 이 정수들과 슬라이스들에 의존할 때, 그건 매우 특정한 구현에 의존하는 것입니다. 이것은 이득을 생각할 때 넘겨 버릴 수 있는, 일정한 부담입니다. 이 경우에서는 비용이 없다고 할 수 있습니다; 만약 정수들과 슬라이스들이 오류가 있다면, *모두가* 오류가 있는 거니까요. 게다가 그것들은 `BTreeMap` 을 관리하는 사람들의 손에 관리되기 때문에, 그 구현들을 지켜보기 쉽죠.

반면에 `BTreeMap` 의 키 타입은 제네릭입니다. 그 `Ord` 구현을 믿는 것은 과거, 현재, 미래의 모든 `Ord` 구현을 믿는 것과 같습니다. 
여기서의 비용은 큽니다: 어디서 누군가는 실수를 해서 본인의 `Ord` 구현을 망치거나, 심지어는 "그냥 되는 것처럼 보여서" 완전한 순서를 가지는 것처럼 거짓말을 할 수도 있습니다. 
그런 일이 벌어질 때 `BTreeMap` 은 대비해야 합니다.



The same logic applies to trusting a closure that's passed to you to behave
correctly.

This problem of unbounded generic trust is the problem that `unsafe` traits
exist to resolve. The `BTreeMap` type could theoretically require that keys
implement a new trait called `UnsafeOrd`, rather than `Ord`, that might look
like this:

```rust
use std::cmp::Ordering;

unsafe trait UnsafeOrd {
    fn cmp(&self, other: &Self) -> Ordering;
}
```

Then, a type would use `unsafe` to implement `UnsafeOrd`, indicating that
they've ensured their implementation maintains whatever contracts the
trait expects. In this situation, the Unsafe Rust in the internals of
`BTreeMap` would be justified in trusting that the key type's `UnsafeOrd`
implementation is correct. If it isn't, it's the fault of the unsafe trait
implementation, which is consistent with Rust's safety guarantees.

The decision of whether to mark a trait `unsafe` is an API design choice. A
safe trait is easier to implement, but any unsafe code that relies on it must
defend against incorrect behavior. Marking a trait `unsafe` shifts this
responsibility to the implementor. Rust has traditionally avoided marking
traits `unsafe` because it makes Unsafe Rust pervasive, which isn't desirable.

`Send` and `Sync` are marked unsafe because thread safety is a *fundamental
property* that unsafe code can't possibly hope to defend against in the way it
could defend against a buggy `Ord` implementation. Similarly, `GlobalAllocator`
is keeping accounts of all the memory in the program and other things like
`Box` or `Vec` build on top of it. If it does something weird (giving the same
chunk of memory to another request when it is still in use), there's no chance
to detect that and do anything about it.

The decision of whether to mark your own traits `unsafe` depends on the same
sort of consideration. If `unsafe` code can't reasonably expect to defend
against a broken implementation of the trait, then marking the trait `unsafe` is
a reasonable choice.

As an aside, while `Send` and `Sync` are `unsafe` traits, they are *also*
automatically implemented for types when such derivations are provably safe
to do. `Send` is automatically derived for all types composed only of values
whose types also implement `Send`. `Sync` is automatically derived for all
types composed only of values whose types also implement `Sync`. This minimizes
the pervasive unsafety of making these two traits `unsafe`. And not many people
are going to *implement* memory allocators (or use them directly, for that
matter).

This is the balance between Safe and Unsafe Rust. The separation is designed to
make using Safe Rust as ergonomic as possible, but requires extra effort and
care when writing Unsafe Rust. The rest of this book is largely a discussion
of the sort of care that must be taken, and what contracts Unsafe Rust must uphold.

[`Send`]: https://doc.rust-lang.org/std/marker/trait.Send.html
[`Sync`]: https://doc.rust-lang.org/std/marker/trait.Sync.html
[`GlobalAlloc`]: https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html
[`SliceIndex`]: https://doc.rust-lang.org/std/slice/trait.SliceIndex.html
[conversions]: conversions.html
[ptr_offset]: https://doc.rust-lang.org/std/primitive.pointer.html#method.offset
[get_unchecked]: https://doc.rust-lang.org/std/primitive.slice.html#method.get_unchecked
[transmute]: https://doc.rust-lang.org/std/mem/fn.transmute.html
[`PartialOrd`]: https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html
[`Ord`]: https://doc.rust-lang.org/std/cmp/trait.Ord.html
[`BTreeMap`]: https://doc.rust-lang.org/std/collections/struct.BTreeMap.html
