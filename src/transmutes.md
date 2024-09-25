# 변질

저리 비켜 타입 시스템! 우린 이 비트들을 재해석하거나 죽을 것이다! 비록 이 책이 불안전한 것들을 하는 것에 대한 내용이지만, 이 섹션에 소개된 내용 말고 **다른 방법을** 깊게 생각해 봐야 한다고 충분히 강조할 수가 없네요. 
이것은 진짜, 진실로, 러스트에서 할 수 있는 가장 끔찍하게 불안전한 것입니다. The guardrails here are dental floss.

[`mem::transmute<T, U>`][transmute]는 `T` 타입의 값을 받아서 `U` 타입의 값이 되도록 재해석합니다. 유일한 제한은 `T`와 `U`가 같은 크기를 가지는 것이 보장되어야 한다는 겁니다. 
이것으로 미정의 동작을 일으키는 방법들은 충격적입니다.

* 첫번째로, 그리고 가장 중요하게 말할 것은, *어떤* 타입의 값이든 올바르지 않은 상태로 만드는 것은 정말로 예상할 수 없는 변덕스러운 혼돈을 초래할 것입니다. `3`을 `bool`로 변질하지 마세요. 그 `bool`로 *아무것도* 하지 않더라도요. 그냥 하지 마세요.

* 변질은 반환 타입이 타입 변수입니다. 만약 반환 타입을 명시하지 않는다면 타입 추론을 만족하기 위해 이상한 타입을 반환할 수 있습니다.

* `&`를 `&mut`로 변질하는 것은 **미정의 동작입니다**. 이것을 사용하는 몇몇 부분이 안전하게 *보일 수* 있지만, 러스트가 최적화를 할 때 불변 레퍼런스는 그 수명 동안 변하지 않는다는 것이라 가정하고, 이런 변질은 그런 가정과 정면으로 충돌할 수 있다는 것을 주의하세요. 따라서:
  * `&`를 `&mut`로 변질하는 것은 *언제나* **미정의 동작입니다**.
  * 안됩니다, 하면 안돼요.
  * 아뇨, 당신은 특별하지 않습니다.

* 레퍼런스로 변질할 때 명확히 수명을 제시하지 않으면 [무제한 수명이][unbounded lifetime] 됩니다.

* 서로 다른 복합 타입들 간에 변질할 때, 타입들이 똑같이 정렬되어 있다는 것을 확실히 해야 합니다! 만약 다르게 정렬되어 있다면, 잘못된 필드가 잘못된 값으로 채워지고, 그러면 당신을 불행하게 만들고 또한 **미정의 동작이** 발생할 수 있습니다 (위를 보세요).

  그럼 똑같이 정렬되어 있는지 어떻게 알까요? `repr(C)` 타입과 `repr(transparent)` 타입들에 대해서는, 어떻게 정렬되는지가 정확하게 정의되어 있습니다. 하지만 당신의 흔해 빠진 `repr(Rust)` 타입은 그렇지가 않습니다.
  같은 제네릭 타입의 다른 인스턴스들조차도 완전히 다르게 정렬될 수 있습니다. `Vec<i32>`와 `Vec<u32>`는 필드들을 같은 순서로 배치했을 *수도* 있고, 아닐 수도 있습니다. 

  So how do you know if the layouts are the same? For `repr(C)` types and
  `repr(transparent)` types, layout is precisely defined. But for your
  run-of-the-mill `repr(Rust)`, it is not. Even different instances of the same
  generic type can have wildly different layout. `Vec<i32>` and `Vec<u32>`
  *might* have their fields in the same order, or they might not. The details of
  what exactly is and is not guaranteed for data layout are still being worked
  out over [at the UCG WG][ucg-layout].

[`mem::transmute_copy<T, U>`][transmute_copy] somehow manages to be *even more*
wildly unsafe than this. It copies `size_of<U>` bytes out of an `&T` and
interprets them as a `U`.  The size check that `mem::transmute` has is gone (as
it may be valid to copy out a prefix), though it is Undefined Behavior for `U`
to be larger than `T`.

Also of course you can get all of the functionality of these functions using raw
pointer casts or `union`s, but without any of the lints or other basic sanity
checks. Raw pointer casts and `union`s do not magically avoid the above rules.

[unbounded lifetime]: ./unbounded-lifetimes.md
[transmute]: https://doc.rust-lang.org/std/mem/fn.transmute.html
[transmute_copy]: https://doc.rust-lang.org/std/mem/fn.transmute_copy.html
[ucg-layout]: https://rust-lang.github.io/unsafe-code-guidelines/layout.html
