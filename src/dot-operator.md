# 점 연산자

점 연산자는 타입을 변환하기 위해 많은 마법을 사용할 겁니다. 타입이 맞을 때까지 자동 레퍼런싱, 자동 역참조, 강제 변환을 수행하겠죠. 메서드 조회의 자세한 작동 방식은 [여기에][method_lookup] 정의되어 있지만, 기본적인 절차를 여기서 간단하게 설명하겠습니다.

어떤 수신자(`self`, `&self` 또는 `&mut self` 매개변수)가 있는 함수 `foo`가 있다고 해 봅시다. 만약 우리가 `value.foo()`를 호출하면, 컴파일러는 이 함수의 올바른 구현을 호출하기 전에 `Self`가 어떤 타입인지 밝혀내야 합니다. 이 예제에서는 `value`가 `T` 타입이라고 하겠습니다.

우리는 [완전 정식화 문법][fqs]을 써서 우리가 어떤 타입의 함수를 호출하는지를 명확히 하겠습니다.

- 먼저 컴파일러는 `T::foo(value)`를 직접 호출할 수 있는지 검사합니다. 이것은 "값에 의한" 메서드 호출이라 부릅니다.
- 이 함수를 호출할 수 없다면 (예를 들어 함수가 잘못된 타입을 가진다거나 `Self`에 대해 트레잇이 구현되지 않았을 경우), 컴파일러는 자동으로 레퍼런스를 추가합니다. 이 말은 컴파일러가 `<&T>::foo(value)`와 `<&mut T>::foo(value)`를 시도해 본다는 뜻입니다. 이것은 자동 레퍼런스 메서드 호출이라 부릅니다.
- 여기까지의 후보들이 실패했다면 컴파일러는 `T`를 역참조하여 다시 시도합니다. 이것은 `Deref` 트레잇을 사용합니다 - 만약 `T: Deref<Target = U>`라면 `T` 대신 `U`가 사용됩니다. 만약 `T`를 역참조할 수 없다면 `T`의 _크기 지정 해제를_ 시도할 수도 있습니다. 이것은 만약 `T`가 컴파일 시간에 알려진 크기 매개변수가 있다면, 메서드를 찾기 위해 이것을 "잊어버린다는" 말입니다. 예를 들어, 이런 크기 지정 해제 작업은 배열의 크기를 "잊어버림으로써" `[i32; 2]`를 `[i32]`로 변환할 수 있습니다.


여기 메서드 조회 알고리즘의 예제가 있습니다:

```rust,ignore
let array: Rc<Box<[T; 3]>> = ...;
let first_entry = array[0];
```

배열로 가는 길에 돌아가는 길이 이렇게 많은데 컴파일러는 어떻게 실제 `array[0]`을 계산할 수 있을까요? 먼저, `array[0]`은 그냥 [`Index`][index] 트레잇을 위한 문법적 설탕입니다 - 컴파일러는 `array[0]`을 `array.index(0)`으로 변환할 겁니다. 
이제, 컴파일러는 함수를 호출하기 위해 `array`가 `Index`를 구현하는지 봅니다.

그럼 컴파일러는 `Rc<Box<[T; 3]>>`가 `Index`를 구현하는지 보는데, 구현하지 않고, `&Rc<Box<[T; 3]>>`나 `&mut Rc<Box<[T; 3]>>`도 `Index`를 구현하지 않습니다. 
여태까지 아무것도 맞지 않았으니, 컴파일러는 `Rc<Box<[T; 3]>>`를 `Box<[T; 3]>`로 역참조하여 다시 시도합니다. `Box<[T; 3]>`, `&Box<[T; 3]>`, 그리고 `&mut Box<[T; 3]>`는 `Index`를 구현하지 않으므로, 
컴파일러는 다시 역참조합니다. `[T; 3]`과 그의 자동 참조들도 `Index`를 구현하지 않습니다. 컴파일러는 `[T; 3]`를 역참조할 수 없으므로, 크기 지정을 해제하여, `[T]`를 얻어냅니다. 마지막으로, `[T]`는 `Index`를 구현하므로, 
컴파일러는 실제로 `index` 함수를 호출할 수 있게 됩니다.

점 연산자가 작동하는 좀더 복잡한 다음의 예제를 생각해 봅시다:

```rust
fn do_stuff<T: Clone>(value: &T) {
    let cloned = value.clone();
}
```

`cloned`는 어떤 타입일까요? 먼저, 컴파일러는 값으로 호출할 수 있는지 알아봅니다. `value`의 타입은 `&T`이고, `clone` 함수는 `fn clone(&T) -> T`의 시그니처를 가지고 있습니다. 컴파일러는 `T: Clone`을 알고 있으니, 
`cloned: T`인 것을 찾아냅니다.

만약 `T: Clone` 제한이 없어졌다면 무슨 일이 일어날까요? `T`를 위한 `Clone` 구현이 없으므로, 컴파일러는 값으로 호출하지 못할 것입니다. 따라서 컴파일러는 자동 참조로 호출을 시도합니다. 이 경우에는 `Self = &T`이므로 
함수는 `fn clone(&&T) -> &T`의 시그니처를 가지게 됩니다. 컴파일러는 `&T: Clone`을 알아차리고, `cloned: &T`라고 결론짓습니다.

여기, 자동 참조 동작이 잘 보이지 않는 변화를 만들어내는 데 쓰이는, 다른 예제가 있습니다.

```rust
use std::sync::Arc;

#[derive(Clone)]
struct Container<T>(Arc<T>);

fn clone_containers<T>(foo: &Container<i32>, bar: &Container<T>) {
    let foo_cloned = foo.clone();
    let bar_cloned = bar.clone();
}
```

`foo_cloned`와 `bar_cloned`는 어떤 타입일까요? 우리는 `Container<i32>: Clone`이라는 것을 알기 때문에, 컴파일러는 `clone`을 값으로 호출하여 `foo_cloned: Container<i32>`를 얻어냅니다. 그러나, 
`bar_cloned`는 실제로는 `&Container<T>`를 타입으로 가지게 됩니다. 확실히 이것은 말이 되지 않습니다 - 우리는 `Container`에 `#[derive(Clone)]`을 추가했으므로, `Container`는 `Clone`을 구현해야 합니다! 
좀더 가까이 보자면, `derive` 매크로에 의해 생성된 코드는 (대강) 다음과 같습니다:

```rust,ignore
impl<T> Clone for Container<T> where T: Clone {
    fn clone(&self) -> Self {
        Self(Arc::clone(&self.0))
    }
}
```



The derived `Clone` implementation is [only defined where `T: Clone`][clone],
so there is no implementation for `Container<T>: Clone` for a generic `T`.
The compiler then looks to see if `&Container<T>` implements `Clone`, which it does.
So it deduces that `clone` is called by autoref, and so `bar_cloned` has type
`&Container<T>`.

We can fix this by implementing `Clone` manually without requiring `T: Clone`:

```rust,ignore
impl<T> Clone for Container<T> {
    fn clone(&self) -> Self {
        Self(Arc::clone(&self.0))
    }
}
```

Now, the type checker deduces that `bar_cloned: Container<T>`.

[fqs]: https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#fully-qualified-syntax-for-disambiguation-calling-methods-with-the-same-name
[method_lookup]: https://rustc-dev-guide.rust-lang.org/method-lookup.html
[index]: https://doc.rust-lang.org/std/ops/trait.Index.html
[clone]: https://doc.rust-lang.org/std/clone/trait.Clone.html#derivable
