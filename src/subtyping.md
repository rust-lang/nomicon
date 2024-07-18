# 부분타입 다형성과 변성(變性, Variance)

러스트는 빌림과 소유권 사이의 관계를 추적하기 위해 수명을 사용합니다. 하지만 수명의 순진한 구현은 너무 제한적이거나, 아니면 미정의 동작을 허용하게 됩니다.

수명을 유연하게 사용하면서도 수명의 오용을 방지하기 위해서, 러스트는 **부분타입 다형성** 과 **변성(變性, Variance)** 을 사용합니다.

예제와 함께 시작해 보죠.

```rust
// 주의: debug는 수명이 *같은* 두 개의 매개변수를 기대합니다.
fn debug<'a>(a: &'a str, b: &'a str) {
    println!("a = {a:?} b = {b:?}");
}

fn main() {
    let hello: &'static str = "hello";
    {
        let world = String::from("world");
        let world = &world; // 'world 는 'static 보다 짧은 수명입니다
        debug(hello, world);
    }
}
```

보수적인 수명의 구현에서는 `hello`와 `world`는 다른 수명을 가지고 있으므로, 우리는 다음과 같은 오류를 볼지도 모릅니다:

```text
error[E0308]: mismatched types
 --> src/main.rs:10:16
   |
10 |         debug(hello, world);
   |                      ^
   |                      |
   |                      expected `&'static str`, found struct `&'world str`
```

이것은 뭔가 부적절할 것입니다. 이 경우에 우리가 원하는 것은 *최소한* `'world`만큼만 사는 타입은 모두 받는 것입니다. 우리의 수명들에 부분타입 다형성을 이용해 봅시다.

## 부분타입 다형성

부분타입 다형성은 한 타입이 다른 타입 대신에 쓰일 수 있다는 개념입니다.

`Sub`이라는 타입이 `Super`라는 타입의 부분타입이라고 해 봅시다 (우리는 이 단원에서 이것을 `Sub <: Super`라고 표현하는 표기법을 사용하겠습니다).

이것이 우리에게 나타내는 것은 `Super`가 정의하는 *요구사항들*의 집합을 `Sub`이 완벽하게 충족한다는 것입니다. 그 다음 `Sub`은 더 많은 요구사항을 가질 수 있겠죠.

이제, 부분타입 다형성을 수명에 쓰기 위해, 우리는 수명의 요구사항을 정의해야 합니다:

> `'a`는 코드 구역을 정의한다.

이제 수명을 위한 요구사항을 만들었으니, 우리는 수명들이 서로 어떻게 관련이 있는지를 정의할 수 있습니다:

> `'long`이 정의하는 코드 구역이 `'short`가 정의하는 구역을 **완전히 포함할 때**, 그리고 오직 그 경우에만 `'long <: 'short`이다.

`'long`은 `'short`가 정의한 구역보다 더 넓은 코드 구역을 정의할 수 있지만, 그래도 우리의 정의에 어긋나지 않습니다.

우리가 이 단원의 나머지를 통해서 보겠지만, 부분타입 다형성은 이것보다는 훨씬 복잡하고 세밀하지만, 이 간단한 규칙은 직관상 99%로 아주 좋습니다.
그리고 만약 불안전한 코드를 작성하지 않는다면, 컴파일러가 당신을 위해 온갖 특수한 경우를 다 처리해 줄 겁니다. 하지만 이것은 러스토노미콘이죠. 우리는 불안전한 코드를 작성할 것이니,
우리는 이것이 실제로 어떻게 동작하는지, 그리고 우리가 이것을 어떻게 가지고 놀 수 있을지를 이해해야 합니다.

위의 예제로 돌아오면, 우리는 `'static <: 'world`라고 말할 수 있습니다. 지금으로써는, 수명의 부분타입 관계가 레퍼런스에도 그대로 전달된다는 것을 일단은 받아들입시다 (더 자세한 건 [변성](#변성變性-variance)에서 다룹니다).
예를 들어, `&'static str`은 `&'world str`의 부분타입이므로, 우리는 `&'static str`을 `&'world str`로 "격하시킬" 수 있습니다. 이렇게 하면, 위의 예제는 컴파일될 겁니다:

```rust
fn debug<'a>(a: &'a str, b: &'a str) {
    println!("a = {a:?} b = {b:?}");
}

fn main() {
    let hello: &'static str = "hello";
    {
        let world = String::from("world");
        let world = &world; // 'world 는 'static 보다 짧은 수명입니다.
        debug(hello, world); // hello 는 조용히 `&'static str`을 `&'world str`로 격하시킵니다.
    }
}
```

## 변성(變性, Variance)

위에서 우리는 `'static <: 'b`가 `&'static T <: &'b T`를 함의한다는 것을 대충 넘어갔었습니다. 이것은 *변성*이라고 알려진 속성을 사용한 것인데요. 이 예제처럼 간단하지만은 않습니다. 이것을 이해하기 위해, 
이 예제를 조금 확장해 보죠:

```rust,compile_fail,E0597
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut hello: &'static str = "hello";
    {
        let world = String::from("world");
        assign(&mut hello, &world);
    }
    println!("{hello}"); // 해제 후 사용 😿
}
```

`assign`에서 우리는 `hello` 레퍼런스를 `world`를 향해 가리키도록 합니다. 하지만 그 다음 `world`는, 나중에 `hello`가 `println!`에서 사용되기 전에, 구역 밖으로 벗어나고 맙니다.

이것은 전형적인 "해제 후 사용" 버그입니다!

우리의 본능은 먼저 `assign`의 구현을 나무랄 수도 있겠지만, 여기에는 잘못된 것이 없습니다. 우리가 `T` 타입의 값을 `T` 타입에 할당하는 것이 그렇게 무리는 아닐 겁니다.

문제는 우리가 `&mut &'static str`과 `&mut &'b str`이 서로 호환되는지를 짐작할 수 없다는 점입니다. 이것이 의미하는 것은 `&mut &'static str`이 `&mut &'b str`의 부분타입이 될 수 **없다는** 말입니다, 
비록 `'static`이 `'b`의 부분타입이라고 해도요.

변성은 제네릭 매개변수를 통한 부분타입들간의 관계를 정의하기 위해 러스트가 빌린 개념입니다.

> 주의: 편의를 위해 우리는 제네릭 타입을 `F<T>`로 정의하여 `T`에 대해 쉽게 말할 것입니다. 이것이 문맥에서 잘 드러나길 바랍니다.

타입 `F`의 *변성*은 그 입력들의 부분타입 다형성이 출력들의 부분타입 다형성에 어떻게 영향을 주느냐 하는 것입니다. 러스트에서는 세 가지 종류의 변성이 있습니다. 두 타입 `Sub`과 `Super`가 있고, `Sub`이 `Super`의 부분타입일 때:

* `F<Sub>`이 `F<Super>`의 부분타입일 경우 `F`는 **공변(共變)합니다** (부분타입 특성이 전달됩니다)
* `F<Super>`가 `F<Sub>`의 부분타입일 경우 `F`는 **반변(反變)합니다** (부분타입 특성이 "뒤집힙니다")
* 그 외에는 `F`는 **무변(無變)합니다** (부분타입 관계가 존재하지 않습니다)

우리가 위의 예제에서 기억한다면, `'a <: 'b`일 경우 `&'a T`를 `&'b T`의 부분타입으로 다뤄도 되었으니, `&'a T`는 `'a`에 대해서 *공변하는* 것이군요.

또한, 우리는 `&mut &'a U`를 `&mut &'b U`의 부분타입으로 다루면 안된다는 것을 보았으니, `&mut T`는 `T`에 대해서 *무변하다고* 말할 수 있겠습니다.

여기 다른 제네릭 타입들과 그들의 변성에 대한 표입니다:

|                 |     'a    |         T         |     U     |
|-----------------|:---------:|:-----------------:|:---------:|
| `&'a T `        | 공변       | 공변               |           |
| `&'a mut T`     | 공변       | 무변               |           |
| `Box<T>`        |           | 공변               |           |
| `Vec<T>`        |           | 공변               |           |
| `UnsafeCell<T>` |           | 무변               |           |
| `Cell<T>`       |           | 무변               |           |
| `fn(T) -> U`    |           | **반**변           | 공변       |
| `*const T`      |           | 공변               |           |
| `*mut T`        |           | 무변               |           |

이 중의 몇 가지는 다른 것들과의 관계로 설명할 수 있습니다:

* `Vec<T>`와 다른 모든 소유하는 포인터들과 컬렉션들은 `Box<T>`와 같은 논리를 따릅니다
* `Cell<T>`와 다른 모든 내부 가변성이 있는 타입들은 `UnsafeCell<T>`와 같은 논리를 따릅니다
* `UnsafeCell<T>`는 내부 가변성이 있으므로 `&mut T`와 같은 변성을 가지게 됩니다
* `*const T`는 `&T`와 같은 논리를 따릅니다
* `*mut T`는 `&mut T`(또는 `UnsafeCell<T>`)와 같은 논리를 따릅니다

더 많은 타입에 대해서는 참조서의 ["Variance" 섹션을][variance-table] 보세요.

[variance-table]: https://doc.rust-lang.org/reference/subtyping.html#variance

> 주의: 러스트 언어에서 반변 타입의 *유일한* 예는 함수의 매개변수이고, 따라서 실제 상황에서는 크게 와닿지 않습니다.
> 반변성을 끌어내려면 특정 수명을 가지고 있는 레퍼런스를 매개변수로 받는 함수 포인터를 가지고 고차원적인 프로그래밍을 해야 합니다
> (만약 "아무 수명"을 모두 받는 레퍼런스였다면, 상계 수명을 이용하게 되는데, 이것은 부분타입 다형성과 독립적으로 작동하기 때문입니다).

이제 우리가 변성에 대한 좀 더 정식적인 이해를 했으니, 더 많은 예제를 더 자세히 살펴봅시다.

```rust,compile_fail,E0597
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut hello: &'static str = "hello";
    {
        let world = String::from("world");
        assign(&mut hello, &world);
    }
    println!("{hello}");
}
```

이것을 실행하면 어떤 결과가 나오나요?

```text
error[E0597]: `world` does not live long enough
  --> src/main.rs:9:28
   |
6  |     let mut hello: &'static str = "hello";
   |                    ------------ type annotation requires that `world` is borrowed for `'static`
...
9  |         assign(&mut hello, &world);
   |                            ^^^^^^ borrowed value does not live long enough
10 |     }
   |     - `world` dropped here while still borrowed
```

다행이군요, 컴파일되지 않습니다! 여기서 무슨 일이 일어나고 있는 건지 자세하게 쪼개봅시다.

먼저 `assign` 함수를 봅시다:

```rust
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}
```

이것이 하는 일은 가변 레퍼런스와 값을 받아서 가변 레퍼런스의 원본을 그 값으로 바꿔치기하는 것밖에 없습니다. 이 함수에 대해 중요한 것은 이 함수가 타입 동치 제약을 만든다는 점입니다. 
이 함수는 시그니처에서 레퍼런스의 원본과 값은 *아주 똑같은* 타입이어야 한다고 명시하고 있습니다.

한편 우리는 이 함수에 `&mut &'static str`과 `&'world str`을 전달합니다.

`&mut T`가 `T`에 대해서 무변하기 때문에, 컴파일러는 첫째 매개변수에 아무런 부분타입 관계도 적용할 수 없다고 결론짓고, 따라서 `T`는 정확히 `&'static str`이어야만 하게 됩니다.

이것은 `&T`의 경우와 반대입니다:

```rust
fn debug<T: std::fmt::Debug>(a: T, b: T) {
    println!("a = {a:?} b = {b:?}");
}
```

여기도 비슷하게 `a`와 `b`는 같은 타입 `T`를 가져야만 하는군요. 하지만 `&'a T`가 `'a`에 대해서 공변*하기* 때문에, 우리는 부분타입 변환을 할 수 있습니다. 
따라서 컴파일러는 `&'static str`이 `&'b str`의 부분타입인 경우에, 그리고 오직 그 경우에만, `&'static str`은 `&'b str`이 될 수 있다고 결정합니다. 
이것은 `'static <: 'b`이면 성립할 텐데, 이 조건은 참이므로, 컴파일러는 행복하게 이 코드의 컴파일을 계속하게 됩니다.

보시다 보면 알겠지만, 왜 `Box`(와 `Vec`, `HashMap`, 등등)가 공변해도 괜찮은지는 수명이 왜 공변해도 괜찮은지와 비슷합니다: 
당신이 이것들에 가변 레퍼런스 같은 것을 끼워넣으려고 한다면, 그들은 무변성을 상속받고 당신은 안 좋은 짓을 하는 것에서 방지될 테니까요.

> 한편 `Box`는 우리가 그냥 지나쳤던, 레퍼런스의 값의 측면에 집중하기 쉽게 해 줍니다.
>
> 값의 레퍼런스들이 얼마든지 복제되어서 자유롭게 읽고 쓸 수 있게 하는 많은 언어들과 달리, 러스트는 매우 엄격한 규칙이 있습니다: 당신이 값을 변경하거나 이동할 수 있다면, 당신이 접근 권한을 가진 유일한 사람이라는 뜻입니다.
>
> 다음 코드를 생각해 봅시다:
> ```rust,ignore
> let hello: Box<&'static str> = Box::new("hello");
> 
> let mut world: Box<&'b str>;
> world = hello;
> ```
> 우리가 `hello`가 `'static` 동안 살아 있었다는 것을 잊은 것은 아무런 문제가 되지 않습니다, 왜냐면 우리가 `hello`를 `'b`동안만 살아 있다고 알고 있는 변수에 옮겼을 때,
> **우리는 그것이 더 오래 살았다고 우주에서 유일하게 알고 있던 것을 없앴기 때문입니다!**

이제 설명할 것이 하나만 남았군요: 함수 포인터입니다.

`fn(T) -> U`가 왜 `U`에 대해서 공변해야 하는지 보기 위해, 다음의 시그니처를 생각해 봅시다:

<!-- ignore: simplified code -->
```rust,ignore
fn get_str() -> &'a str;
```

이 함수는 어떤 수명 `'a`에 묶인 `str`을 생산한다고 주장합니다. 그런 의미에서, 대신 이런 시그니처의 함수를 제공해도 완벽히 유효합니다:

<!-- ignore: simplified code -->
```rust,ignore
fn get_static() -> &'static str;
```

이 함수를 호출할 때, 반환되길 기대하는 값은 최소한 `'a`만큼 사는 `&str`이니, 실제로는 값이 더 살아도 상관 없겠죠.

그러나, *매개변수들에는* 같은 논리가 통하지 않습니다. 이 조건을:

<!-- ignore: simplified code -->
```rust,ignore
fn store_ref(&'a str);
```

이것으로 만족시켜 보려고 생각해 보세요:

<!-- ignore: simplified code -->
```rust,ignore
fn store_static(&'static str);
```

첫번째 함수는 최소한 `'a`만큼만 산다면 아무 문자열 레퍼런스를 받을 수 있지만, 두번째는 `'static`보다 짧게 사는 문자열 레퍼런스를 받을 수 없으니, 이것은 갈등을 초래하겠군요. 여기서는 공변성이 통하지 않습니다. 
하지만 이것을 반대로 생각하면, 말이 *됩니다!* 만약 우리가 `&'static str`을 받는 함수가 필요하다면, *아무* 레퍼런스 수명이나 받는 함수는 잘 동작할 겁니다.

실전에서 한번 살펴보죠.

```rust,compile_fail
# use std::cell::RefCell;
thread_local! {
    pub static StaticVecs: RefCell<Vec<&'static str>> = RefCell::new(Vec::new());
}

/// 주어진 input을 스레드 지역변수 `Vec<&'static str>`에 집어넣습니다
fn store(input: &'static str) {
    StaticVecs.with_borrow_mut(|v| v.push(input));
}

/// 함수와 입력값을 받아서 입력값을 함수에 호출합니다 (같은 수명이어야 합니다!)
fn demo<'a>(input: &'a str, f: fn(&'a str)) {
    f(input);
}

fn main() {
    demo("hello", store); // "hello"는 'static입니다. `store`를 문제없이 호출할 수 있죠.

    {
        let smuggle = String::from("smuggle");

        // `&smuggle`은 'static이 아닙니다. 만약 우리가 `store`에 `&smuggle`을 전달하면,
        // `StaticVecs`에 잘못된 수명을 집어넣어 버린 게 될 겁니다.
        // 따라서, `fn(&'static str)`은 `fn(&'a str)`의 부분타입이 될 수 없습니다.
        demo(&smuggle, store);
    }

    // 해제 후 사용 😿
    StaticVecs.with_borrow(|v| println!("{v:?}"));
}
```

그리고 이것이 다른 타입들과 달리, 함수 타입들이 그 매개변수들에 대해서 **반**변하는 이유입니다.

자, 이제 표준 라이브러리가 제공하는 타입들은 잘 살펴보았는데, *당신이* 정의한 타입들의 변성은 어떨까요? 간단하게 말하자면, 구조체는 그 필드들의 변성을 상속받습니다. 
만약 `MyType` 구조체가 필드 `a`에 쓰이는 제네릭 매개변수 `A`를 가지고 있다면, `A`에 대한 `MyType`의 변성은 `A`에 대한 `a`의 변성과 똑같습니다.

하지만 만약 `A`가 여러 필드에 쓰인다면:

* `A`를 사용하는 모든 타입이 공변한다면, `MyType`은 `A`에 대해서 공변합니다
* `A`를 사용하는 모든 타입이 반변한다면, `MyType`은 `A`에 대해서 반변합니다
* 그 외에는, `MyType`은 `A`에 대해서 무변합니다

```rust
use std::cell::Cell;

struct MyType<'a, 'b, A: 'a, B: 'b, C, D, E, F, G, H, In, Out, Mixed> {
    a: &'a A,     // 'a와 A에 대해서 공변합니다
    b: &'b mut B, // 'b에 대해서 공변하고 B에 대해서 무변합니다

    c: *const C,  // C에 대해서 공변합니다
    d: *mut D,    // D에 대해서 무변합니다

    e: E,         // E에 대해서 공변합니다
    f: Vec<F>,    // F에 대해서 공변합니다
    g: Cell<G>,   // G에 대해서 무변합니다

    h1: H,        // 원래대로라면 H에 대해서 공변하겠지만...
    h2: Cell<H>,  // 변성이 충돌하면 무변성이 이기기 때문에, H에 대해서 무변하게 됩니다

    i: fn(In) -> Out,       // In에 대해서 반변하고, Out에 대해서 공변합니다

    k1: fn(Mixed) -> usize, // 원래대로라면 Mixed에 대해서 반변하겠지만..
    k2: Mixed,              // 변성이 충돌할 경우 무변성이 되기 때문에, Mixed에 대해서 무변하게 됩니다
}
```
