# 해제 검사

우리는 수명이 간단한 규칙으로 우리가 달랑거리는 레퍼런스를 절대 읽지 않도록 보장하는 것을 봤습니다. 하지만 이 동안 우리는 한 수명이 다른 수명보다 *오래 산다고* 할 때, 두 수명이 같은 경우도 포함한 경우만 말했습니다. 즉, 우리가 `'a: 'b`라고 할 때, `'a`가 *딱* `'b`만큼만 살아도 괜찮았다는 말입니다. 처음 보면, 이것은 의미없는 구분 같습니다. 같은 시점에 동시에 해제되는 값은 없잖아요, 그렇죠? 그래서 우리는 이런 `let` 문장들을:

<!-- ignore: simplified code -->
```rust,ignore
let x;
let y;
```

이렇게 해독할 수 있었죠:

<!-- ignore: desugared code -->
```rust,ignore
{
    let x;
    {
        let y;
    }
}
```

이렇게 코드 구역을 이용해서 해독할 수 없는, 좀더 복잡한 상황들도 있지만, 순서는 여전히 정의되어 있습니다 - 변수들은 그 정의 순서의 반대로 해제되고, 구조체와 튜플의 필드들은 그 정의 순서대로 해제됩니다. [RFC 1857][rfc1857]에 해제 순서에 대한 좀더 자세한 내용이 있습니다.

이렇게 해 봅시다:

<!-- ignore: simplified code -->
```rust,ignore
let tuple = (vec![], vec![]);
```

왼쪽 벡터가 먼저 해제됩니다. 하지만 이것이 대여 검사기의 눈에 오른쪽 벡터가 왼쪽 벡터보다 엄밀하게 더 오래 산다는 것을 뜻할까요? 이 질문의 답은 *아니라는 겁니다*. 대여 검사기는 튜플의 필드들을 분리해서 추적할 수 있지만, 벡터 원소들에 대해서는 무엇이 무엇보다 오래 사는지 결정할 수 없을 겁니다, 왜냐하면 그 원소들은 대여 검사기가 이해하지 못하는, 순수한 라이브러리 코드로 해제되기 때문입니다.

그래서 우리는 이것을 왜 신경쓸까요? 우리가 신경쓰는 이유는 만약 타입 시스템이 주의하지 않으면, 우발적으로 달랑거리는 포인터를 만들 수도 있기 때문입니다. 다음의 간단한 프로그램을 생각해 보세요:

```rust
struct Inspector<'a>(&'a u8);

struct World<'a> {
    inspector: Option<Inspector<'a>>,
    days: Box<u8>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days));
}
```

이 프로그램은 완벽히 건전하고 오늘날 컴파일됩니다. `days`가 `inspector`보다 엄밀하게 더 오래 살지는 않는다는 사실이 중요하지 않습니다. `inspector`가 살아있는 동안, `days`도 그럴 테니까요.

그러나 우리가 소멸자를 추가하면, 이 프로그램은 더 이상 컴파일되지 않습니다!

```rust,compile_fail
struct Inspector<'a>(&'a u8);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("I was only {} days from retirement!", self.0);
    }
}

struct World<'a> {
    inspector: Option<Inspector<'a>>,
    days: Box<u8>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days));
    // `days`가 먼저 해제되게 되었다고 가정해 봅시다.
    // 그럼 `Inspector`가 해제될 때, 이미 해제된 메모리를 읽으려고 할 겁니다!
}
```

```text
error[E0597]: `world.days` does not live long enough
  --> src/main.rs:19:38
   |
19 |     world.inspector = Some(Inspector(&world.days));
   |                                      ^^^^^^^^^^^ borrowed value does not live long enough
...
22 | }
   | -
   | |
   | `world.days` dropped here while still borrowed
   | borrow might be used here, when `world` is dropped and runs the destructor for type `World<'_>`
```

필드의 순서를 바꾸거나 구조체 대신 튜플을 쓴다 해도, 컴파일되지는 않을 겁니다.

`Drop`을 구현하는 것은 `Inspector`가 죽는 동안 어떤 임의의 코드를 실행하게 해 줍니다. 이것이 의미하는 것은 `Inspector`가 사는 동안 살기로 되어있는 타입들이 실제로는 먼저 해제되었는지 관찰할 수도 있다는 것입니다.

흥미롭게도, 제네릭 타입들만 이런 것에 대해서 걱정해야 합니다. 제네릭이 아니라면, 그들이 사용할 수 있는 수명은 `'static`밖에 없는데, 이것은 정말로 *영원히* 살 것이기 때문입니다. 이것이 바로 이런 문제가 *건전한 제네릭 해제*라고 불리는 이유입니다. 건전한 제네릭 해제는 *해제 검사기*에 의해 강제됩니다. 이 글을 쓰는 시점에서, 해제 검사기(dropck라고도 부릅니다)가 어떻게 타입을 검증하는지에 대한 자세한 내용은 전혀 정해져 있지 않습니다. 하지만 큰 틀에서의 규칙은 우리가 이 섹션 내내 집중해온 엄밀함입니다:

**제네릭 타입이 건전하게 `Drop`을 구현하려면, 그 제네릭 매개변수들이 엄밀하게 더 오래 살아야 합니다.**

이 규칙을 지키는 것은 (보통은) 대여 검사기를 만족시키기 위해서 필수적입니다; 이 규칙을 지키는 것은 건전하기에는 충분하지만 필수적이지는 않습니다. 즉, 당신의 타입이 이 규칙을 지킨다면 해제되기에 확실히 건전하다는 말입니다.

이 규칙을 만족시키는 것이 항상 필수는 아닌 이유는 타입이 빌린 데이터를 접근할 수 있음에도, 어떤 `Drop` 구현은 빌린 데이터를 접근하지 않거나, 
아니면 우리가 세부적인 해제 순서를 알고, 그럼에도 빌린 데이터는 괜찮을 것을 알 수도 있기 때문입니다, 비록 대여 검사기가 이를 모르더라도요.

예를 들어, 위의 `Inspector`를 이렇게 변형하면 빌린 데이터를 절대 접근하지 않을 겁니다:

```rust,compile_fail
struct Inspector<'a>(&'a u8, &'static str);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("Inspector(_, {})는 보지 *않아야* 할 때를 압니다.", self.1);
    }
}

struct World<'a> {
    inspector: Option<Inspector<'a>>,
    days: Box<u8>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days, "gadget"));
    // `days`가 먼저 해제되게 된다고 해 봅시다.
    // `Inspector`가 해제되어도, 그 소멸자는 빌린 `days`를
    // 접근하지 않을 겁니다.
}
```

마찬가지로, 이렇게 변형한 것도 빌린 데이터를 절대 접근하지 않을 겁니다:

```rust,compile_fail
struct Inspector<T>(T, &'static str);

impl<T> Drop for Inspector<T> {
    fn drop(&mut self) {
        println!("Inspector(_, {})는 보지 *않아야* 할 때를 압니다.", self.1);
    }
}

struct World<T> {
    inspector: Option<Inspector<T>>,
    days: Box<u8>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days, "gadget"));
    // `days`가 먼저 해제되게 된다고 해 봅시다.
    // `Inspector`가 해제되어도, 그 소멸자는 빌린 `days`를 
    // 접근하지 않을 겁니다.
}
```

하지만, 위의 수정된 코드들은 `fn main`을 분석할 때 *둘 모두* 대여 검사기에게 거부됩니다, "`days`가 충분히 오래 살지 않는다"고 말이죠.

그 이유는 `main`의 대여 검사는 `Inspector`의 `Drop` 구현의 내부는 모르기 때문입니다. 대여 검사기가 `main`을 분석하는 동안 아는 것이라고는, `Inspector`의 소멸자의 본문이 빌린 데이터를 접근할 수도 있다는 점입니다.

따라서, 해제 검사기는 어떤 값 안에 있는 모든 빌린 데이터(레퍼런스가 아닌 본체)가 그 값보다 오래 살도록 강제합니다.

## 탈출 장치

해제 검사를 좌우하는 정확한 규칙들은 미래에는 좀 덜 엄격할 수도 있겠습니다.

현재의 분석은 의도적으로 보수적이고 흔합니다; 어떤 값에 있는 모든 레퍼런스의 본체들이 그 값보다 오래 살도록 강제하는데, 이것은 확실히 건전하기 때문입니다.

언어의 미래 버전들은 분석을 더 예리하게 해서, 건전한 코드가 안전하지 않다고 거부되는 경우들을 줄일 수 있을 것입니다. 그러면 위의 두 `Inspector`들과 같이 소멸하는 동안 그 원소를 접근하지 않는 경우를 해결하는 데 도움이 될 것입니다.

그 동안, 제네릭 타입의 소멸자가 수명이 다한 데이터를 접근하지 않는다고 (불안전하게) *보장하는* 불안정 속성이 있습니다, 비록 그 타입으로는 수명이 다한 데이터를 접근할 수 있더라도요.

이 속성은 `may_dangle`이라 부르고, [RFC 1327][rfc1327]에서 소개되었습니다. 이것을 위의 `Inspector`에 적용하려면, 이렇게 씁니다:

```rust
#![feature(dropck_eyepatch)]

struct Inspector<'a>(&'a u8, &'static str);

unsafe impl<#[may_dangle] 'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("Inspector(_, {})는 보지 *않아야* 할 때를 압니다.", self.1);
    }
}

struct World<'a> {
    days: Box<u8>,
    inspector: Option<Inspector<'a>>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days, "gadget"));
}
```

이 속성을 사용하려면 `Drop` 구현을 `unsafe`로 수식해야 하는데, 이는 수명이 다했을 수 있는 데이터(예를 들어 위의 `self.0`)를 접근하지 않는다는 보장을 컴파일러가 검사하지 않기 때문입니다.

이 속성은 어느 숫자의 수명이나 타입 매개변수에도 적용할 수 있습니다. 다음의 예제에서 우리는 `'b`의 수명을 갖는 레퍼런스를 접근하지 않고, `T`는 오직 이동 혹은 해제만 할 것이라고 컴파일러에게 알립니다. 하지만 `'a`와 `U`에 대해서는 이 속성을 쓰지 않음으로써 우리가 이 수명과 이 타입의 데이터를 접근할 것이라고 알립니다:

```rust
#![feature(dropck_eyepatch)]
use std::fmt::Display;

struct Inspector<'a, 'b, T, U: Display>(&'a u8, &'b u8, T, U);

unsafe impl<'a, #[may_dangle] 'b, #[may_dangle] T, U: Display> Drop for Inspector<'a, 'b, T, U> {
    fn drop(&mut self) {
        println!("Inspector({}, _, _, {})", self.0, self.3);
    }
}
```

보통은 위의 경우처럼, 필드 접근이 불가능하다는 것이 자명할 때가 많습니다. 그러나 제네릭 타입 매개변수를 가지고 작업하다 보면, 그런 접근이 간접적으로 일어날 수 있습니다. 간접적인 접근의 예는 다음과 같습니다:

- 콜백 호출하기
- 트레잇 메서드를 통해서

(`impl` 구체화 같은 러스트의 미래 변화에 따라, 이런 간접적 접근을 가능하게 하는 방법이 더 많아질 수 있습니다.)

여기 콜백을 호출하는 예제가 있습니다:

```rust
struct Inspector<T>(T, &'static str, Box<for <'r> fn(&'r T) -> String>);

impl<T> Drop for Inspector<T> {
    fn drop(&mut self) {
        // 예를 들어 만약 `T`가 `&'a _`라면, `self.2` 호출이 빌린 데이터를 접근할 수 있습니다.
        println!("Inspector({}, {})는 자신도 모르게 파기된 데이터를 봅니다.",
                 (self.2)(&self.0), self.1);
    }
}
```

이것은 트레잇 메서드를 호출하는 예제입니다:

```rust
use std::fmt;

struct Inspector<T: fmt::Display>(T, &'static str);

impl<T: fmt::Display> Drop for Inspector<T> {
    fn drop(&mut self) {
        // 예를 들어 만약 `T`가 `&'a _`이면, 밑에 숨겨진 `<T as Display>::fmt` 호출은
        // 빌린 데이터를 접근할 수 있습니다.
        println!("Inspector({}, {})는 자신도 모르게 파기된 데이터를 봅니다.",
                 self.0, self.1);
    }
}
```

그리고 당연히, 이런 접근들은 소멸자에 의해 호출되는 다른 어떤 메서드 안에 숩겨져 있을 수도 있습니다, 꼭 직접 쓰여지지 않고도요.

`&'a u8`이 소멸자에서 접근되는 위의 모든 경우에서, `#[may_dangle]` 속성을 추가하면 그 타입은 대여 검사기가 잡지 않을 오용을 막기 힘들어지고, 대혼란을 야기할 수 있습니다. 이 속성은 사용하지 않는 게 좋겠네요.

> ### 해제 순서에 관하여 
>
> 구조체 안의 필드의 해제 순서는 정의되어 있지만, 이것에 의존하는 것은 불안정하고 애매합니다.
> 해제 순서가 중요할 때에는, [`ManuallyDrop`] 타입을 대신 쓰는 것이 좋습니다.

## 해제 검사기에 대해서는 이게 전부인가요?

사실 우리가 불안전한 코드를 작성할 때, 보통은 해제 검사기를 전혀 신경쓰지 않아도 됩니다. 하지만 우리가 신경써야 하는 한 가지 특수한 경우가 있는데, 이것에 관해서는 다음 섹션에서 살펴보겠습니다.

[rfc1327]: https://github.com/rust-lang/rfcs/blob/master/text/1327-dropck-param-eyepatch.md
[rfc1857]: https://github.com/rust-lang/rfcs/blob/master/text/1857-stabilize-drop-order.md
[`manuallydrop`]: https://doc.rust-lang.org/std/mem/struct.ManuallyDrop.html
