# 검사받지 않는 미초기화 메모리

이 규칙의 한 흥미로운 예외는 배열을 가지고 작업할 때입니다. 안전한 러스트는 배열을 부분적으로 초기화하도록 허용하지 않습니다. 배열을 초기화하려면, `let x = [val; N]` 과 같이 전부 다 같은 값으로 설정하거나, 
아니면 `let x = [val1, val2, val3]` 과 같이 하나하나 나열해서 할당합니다. 불행하게도 이것은 꽤나 융통성이 없습니다, 특히 배열을 좀더 점진적으로, 또는 동적으로 초기화하고 싶을 때 말이죠.

불안전한 러스트는 이런 문제를 해결하기 위해 강력한 도구를 선사합니다: [`MaybeUninit`]이죠. 이 타입은 아직 완전히 초기화되지 않은 메모리를 다루기 위해 사용될 수 있습니다.

`MaybeUninit` 으로 우리는, 다음과 같이 배열을 원소별로 초기화할 수 있습니다:

```rust
use std::mem::{self, MaybeUninit};

// 배열의 크기는 손으로 적혀 있지만 바꾸기 쉽습니다 (이 상수의 값만 바꾸면 되니까요).
// 하지만 이것은 우리가 배열을 초기화할 때 [a, b, c] 와 같은 문법을 사용할 수 없다는 것을 말합니다,
// `SIZE`가 바뀔 때마다 그 구문이 계속 바뀔 테니까요!
const SIZE: usize = 10;

let x = {
    // `MaybeUninit`의 미초기화된 배열을 만듭니다. `assume_init`은 안전한데,
    // 우리가 여기서 초기화했다고 주장하는 것들은 `MaybeUninit`들인데,
    // 이들은 초기화를 필요로 하지 않기 때문입니다.
    let mut x: [MaybeUninit<Box<u32>>; SIZE] = unsafe {
        MaybeUninit::uninit().assume_init()
    };

    // `MaybeUninit`은 범위 밖으로 벗어나도 아무 일도 일어나지 않습니다.
    // 따라서 `ptr::write` 대신 생 포인터 할당을 이용해도 기존의 미초기화된 값이 해제되지 않습니다.
    // `Box`는 `panic!`할 수 없으므로 예외 안전성은 걱정할 거리가 못 됩니다.
    for i in 0..SIZE {
        x[i] = MaybeUninit::new(Box::new(i as u32));
    }

    // 모든 것이 초기화됐습니다. 배열을 초기화된 타입으로 변질합니다.
    unsafe { mem::transmute::<_, [Box<u32>; SIZE]>(x) }
};

dbg!(x);
```

이 코드는 3가지의 단계로 나아갑니다:

1. `MaybeUninit<T>` 의 배열을 만듭니다. 현재의 러스트 안정 버전으로는 이것을 위해서는 불안전한 코드를 써야 합니다: 우리는 미초기화된 메모리 조각을 가져다가 (`MaybeUninit::uninit()`) 그것을 완전히 초기화했다고 주장합니다 ([`assume_init()`][assume_init]). 이것은 우스꽝스럽게 보입니다, 우리가 이것을 초기화하지는 않았거든요! 이것이 맞는 이유는 배열 전체가 초기화를 필요로 하지 않는 `MaybeUninit` 으로 이루어졌기 때문입니다. 다른 대부분의 타입들에 대해서는, `MaybeUninit::uninit().assume_init()` 은 그 타입의 잘못된 값을 만들어내고, 그러면 **미정의 동작이** 튀어나오겠죠.

2. 배열을 초기화합니다. 이것의 잘 보이지 않는 점은 보통, 우리가 `=`를 사용해서 러스트 타입 검사기가 이미 초기화되었다고 판단한 타입에 값을 할당할 때 (`x[i]` 같은), 좌변에 있던 이전의 값은 해제된다는 겁니다. 이건 재앙이 될 겁니다. 하지만, 이 경우에는 좌변의 타입이 `MaybeUninit<Box<u32>>` 이고, 이것을 해제해 봐야 아무 것도 일어나지 않습니다! 이 `drop` 사항에 관해서는 밑에서 좀더 논의하겠습니다.

3. 마지막으로, 우리는 배열의 타입에서 `MaybeUninit` 을 지워야 합니다. 현재의 안정적인 러스트 버전으로는, 이 작업은 `transmute`를 써야 합니다. 이 변질은 합당한데 이는 메모리 안에서 `MaybeUninit<T>`은 `T`와 똑같아 보이기 때문입니다.


    하지만, 보통은 `Container<MaybeUninit<T>>>`는 `Container<T>`와 똑같아 보이지 *않습니다*! 만약 `Container`가 `Option`이고, `T`가 `bool`이라고 가정할 때,
   `Option<bool>`은 `bool`이 오직 유효한 2개의 값을 가지고 있다는 것을 이용하지만, `Option<MaybeUninit<bool>>`은 `bool`이 초기화되지 않아도 되기 때문에 그런 작업을 할 수 없습니다.

    따라서, `MaybeUninit`을 변질해서 타입에서 지워도 되는지는 `Container`에 따라 다릅니다. 배열에 대해서는 그렇습니다 (그리고 결국 표준 라이브러리도 이것을 알아차리고 적당한 메서드를 제공할 겁니다).

중간에 있는 반복문은 좀더 시간을 들일 가치가 있는데, 특히 할당문과 또한 `drop` 간의 관계가 그렇습니다. 우리가 만약 이런 코드를 쓴다면:

<!-- ignore: simplified code -->
```rust,ignore
*x[i].as_mut_ptr() = Box::new(i as u32); // 틀림!
```

우리는 `Box<u32>`를 실제로 덮어쓰게 되고, 미초기화된 데이터가 `drop`되며, 이는 엄청난 슬픔과 고통으로 다가올 것입니다.

올바른 대체 방법은, 만약 어떤 이유로 우리가 `MaybeUninit::new`를 사용할 수 없다면, [`ptr`] 모듈을 사용하는 것입니다. 
특히 이 모듈은 기존 값을 해제시키지 않으면서 메모리 위치에 값을 할당할 수 있게 해 주는 3개의 함수를 제공합니다: [`write`], [`copy`], 그리고 [`copy_nonoverlapping`]이죠.

* `ptr::write(ptr, val)`는 `val`을 가지고 `ptr`이 가리키는 주소에 옮겨 놓습니다.
* `ptr::copy(src, dest, count)`는 `count`만큼의 `T` 값들이 차지하는 비트들을 `src`에서 `dest`로 복사합니다. (이것은 C의 memmove와 같습니다 -- 다만 매개변수의 순서가 거꾸로입니다!)
* `ptr::copy_nonoverlapping(src, dest, count)`는 `copy`가 하는 일을 하지만, 두 메모리 영역이 겹치지 않는다는 가정 하에 작업하기 때문에 좀더 빠릅니다. (이것은 C의 memcpy와 같습니다 -- 다만 매개변수의 순서가 거꾸로입니다!)

이 함수들이 오용된다면 심각한 피해를 초래하거나 바로 **미정의 동작을** 유발할 거라는 것은 두말할 필요가 없겠죠. 이 함수들 *자체에* 있는 요구사항은 읽고 쓰는 메모리 위치가 메모리가 할당되고 잘 정렬되어 있어야 한다는 것입니다. 
하지만, 임의의 비트들을 임의의 메모리 상의 위치에 씀으로써 프로그램이 망가지는 방법은 정말 셀 수가 없습니다!

`Drop`을 구현하지 않거나 `Drop` 타입들을 포함하지 않는 타입에 `ptr::write` 식의 장난을 치는 것은 걱정할 필요가 없다는 것은 알아 두세요, 러스트는 그것을 알고 그 값들을 해제하지 않을 것이기 때문입니다. 
이것이 바로 위의 예제에서 우리가 근거로 삼았던 사실입니다.

하지만 미초기화된 메모리를 가지고 작업할 때, 위의 것 같이 값들이 완전히 초기화되기 전에 러스트가 값들을 해제하려고 시도하지는 않는지 항상 경계해야 합니다. 
만약 소멸자가 있다면, 그 변수의 모든 프로그램 상의 경우는 그 범위가 끝나기 전에 값을 초기화해야 합니다. *[이것은 코드가 `panic!`하는 것도 포함합니다](unwinding.html)*.
`MaybeUninit`은 여기서 우리를 조금 도와주는데, 암묵적으로 그 내용물을 해제하지 않기 때문입니다 - 
하지만 `panic!`이 일어날 경우 이 모든 것이 의미하는 것은 아직 초기화되지 않은 부분들의 이중 해제 대신, 이미 초기화된 부분들의 메모리 누수로 끝난다는 점입니다.

주의할 것은, `ptr` 메서드들을 사용하려면 우선 초기화하고 싶은 데이터의 *생 포인터를* 얻어내야 합니다. 초기화되지 않은 데이터에 *레퍼런스를* 만드는 것은 불법이고, 따라서 생 포인터를 얻을 때는 주의해야 합니다:

* `T`의 배열에 있어서는, 배열의 인덱스 `idx`번째를 계산할 때는 `base_ptr: *mut T`일 때 `base_ptr.add(idx)`를 사용하면 됩니다. 이것은 메모리에 배열이 어떻게 배치되는지를 이용합니다.
* 하지만 구조체의 경우, 일반적으로 우리는 어떻게 배치되어 있는지 알지 못합니다. 또한 우리는 `&mut base_ptr.field`를 사용할 수 없는데, 레퍼런스를 만드는 행위이기 때문입니다. 따라서, [`addr_of_mut`] 매크로를 조심스럽게 사용해야 합니다. 이것은 중간의 레퍼런스를 만들지 않고 바로 구조체의 필드를 가리키는 생 포인터를 만들어 냅니다:

```rust
use std::{ptr, mem::MaybeUninit};

struct Demo {
    field: bool,
}

let mut uninit = MaybeUninit::<Demo>::uninit();
// `&uninit.as_mut().field` 는 초기화되지 않은 `bool`에 레퍼런스를 만들어낼 겁니다,
// 따라서 **미정의 동작이** 일어납니다!
let f1_ptr = unsafe { ptr::addr_of_mut!((*uninit.as_mut_ptr()).field) };
unsafe { f1_ptr.write(true); }

let init = unsafe { uninit.assume_init() };
```

마지막 당부는, 오래된 러스트 코드를 볼 때, 폐기된 `mem::uninitialized` 함수를 마주칠지도 모릅니다. 이 함수는 스택의 초기화되지 않은 메모리를 처리하는 유일한 방법이었지만, 
언어의 다른 부분과 잘 통합되는 것이 불가능하다고 판단되었습니다. 항상 새로운 코드에서는 그 대신 `MaybeUninit`을 사용하시고, 기회가 있을 때 오래된 코드를 변환하세요.

초기화되지 않은 메모리를 가지고 작업하는 것에 대한 내용은 이 정도쯤 되겠습니다! 기본적으로 어느 곳에 어떤 것이든 초기화되지 않은 메모리가 전달되는 것은 기대하지 않기 때문에, 
만약 조금이라도 초기화되지 않은 메모리를 어딘가에 놓는다면, *매우* 조심하세요.

[`MaybeUninit`]: https://doc.rust-lang.org/core/mem/union.MaybeUninit.html
[assume_init]: https://doc.rust-lang.org/core/mem/union.MaybeUninit.html#method.assume_init
[`ptr`]: https://doc.rust-lang.org/core/ptr/index.html
[`addr_of_mut`]: https://doc.rust-lang.org/core/ptr/macro.addr_of_mut.html
[`write`]: https://doc.rust-lang.org/core/ptr/fn.write.html
[`copy`]: https://doc.rust-lang.org/std/ptr/fn.copy.html
[`copy_nonoverlapping`]: https://doc.rust-lang.org/std/ptr/fn.copy_nonoverlapping.html
