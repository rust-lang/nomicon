# 이량(異量) 타입

거의 항상, 우리는 타입이 정적으로 알려져 있고 양수의 크기를 가지고 있다고 생각합니다. 러스트에서 이것은 항상 그렇지는 않습니다.

## 동량(動量) 타입 (DST)

러스트는 동량(動量) 타입(DST)을 지원합니다: 정적으로 알려진 크기나 정렬선이 없는 타입을 말이죠. 표면적으로는 이것은 좀 말이 되지 않습니다: 러스트는 무언가와 올바르게 작업하기 위해서는 그것의 크기와 정렬선을 *알아야 하거든요!* 
이런 면에서 DST는 보통의 타입이 아닙니다. 정적으로 알려진 크기가 없기 때문에, 이런 타입들은 포인터 뒤에서만 존재할 수 있습니다. 
따라서 DST를 가리키는 포인터는 포인터와 DST를 "완성하는" 정보로 이루어진 *넓은* 포인터가 됩니다 (밑에서 더 설명합니다).

언어에서 보이는 주요한 DST는 두 가지가 있습니다: 

* 트레잇 객체: `dyn MyTrait`
* 슬라이스: [`[T]`][slice], [`str`], 등등

트레잇 객체는 그것이 특정하는 트레잇을 구현하는 어떤 타입을 표현합니다. 정확한 원래 타입은 런타임 리플렉션을 위해 *지워지고,* 타입을 쓰기 위해 필요한 모든 정보를 담고 있는 vtable로 대체됩니다. 트레잇 객체를 완성하는 정보는 이 vtable의 포인터입니다. 포인터가 가리키는 대상의 런타임 크기는 vtable에서 동적으로 요청될 수 있습니다. 

슬라이스는 어떤 연속적인 저장소에 대한 뷰일 뿐입니다 -- 보통 이 저장소는 배열이거나 `Vec`입니다. 슬라이스 포인터를 완성시키는 정보는 가리키고 있는 원소들의 갯수입니다. 
가리키는 대상의 런타임 크기는 그냥 한 원소의 정적으로 알려진 크기와 원소들의 갯수를 곱한 것입니다.

구조체는 사실 마지막 필드로써 하나의 동량 타입을 직접 저장할 수 있지만, 그러면 그들 자신도 동량 타입이 됩니다: 

```rust
// 직접적으로 스택에 저장할 수 없음
struct MySuperSlice {
    info: u32,
    data: [u8],
}
```

이런 타입은 생성할 방법이 없으면 별로 쓸모가 없지만 말이죠. 현재 유일하게 제대로 지원되는, 커스텀 동량 타입을 만들 방법은 타입을 제네릭으로 만들고 *크기 강제 망각*을 실행하는 것입니다: 

```rust
struct MySuperSliceable<T: ?Sized> {
    info: u32,
    data: T,
}

fn main() {
    let sized: MySuperSliceable<[u8; 8]> = MySuperSliceable {
        info: 17,
        data: [0; 8],
    };

    let dynamic: &MySuperSliceable<[u8]> = &sized;

    // 출력: "17 [0, 0, 0, 0, 0, 0, 0, 0]"
    println!("{} {:?}", dynamic.info, &dynamic.data);
}
```

(네, 커스텀 동량 타입은 지금으로써는 매우 설익은 기능입니다.)

## 무량(無量) 타입 (ZST)

러스트는 타입이 공간을 차지하지 않는다고 말하는 것도 허용합니다: 

```rust
struct Nothing; // 필드 없음 = 크기 없음

// 모든 필드가 크기 없음 = 크기 없음
struct LotsOfNothing {
    foo: Nothing,
    qux: (),      // 빈 튜플은 크기가 없습니다
    baz: [u8; 0], // 빈 배열은 크기가 없습니다
}
```

무량(無量) 타입(ZST)은, 당연하게도 그 자체로는, 별로 쓸모가 없습니다. 하지만 러스트의 많은 기이한 레이아웃 선택들이 그렇듯이, 그들의 잠재력은 일반적인 환경에서 빛나게 됩니다: 
러스트는 무량 타입의 값을 생성하거나 저장하는 모든 작업이 아무 작업도 하지 않는 것과 같을 수 있다는 사실을 매우 이해하거든요. 일단 값을 저장한다는 것부터가 말이 안됩니다 -- 차지하는 공간도 없는걸요. 
또 그 타입의 값은 오직 하나이므로, 어떤 값이 읽히든 그냥 무에서 값을 만들어내면 됩니다 -- 이것 또한 차지하는 공간이 없기 때문에, 아무것도 하지 않는 것과 같습니다.

이것의 가장 극단적인 예시 중 하나가 `Map`과 `Set`입니다. `Map<Key, Value>`가 주어졌을 때, `Set<Key>`를 `Map<Key, UselessJunk>`를 적당히 감싸는 자료구조로 만드는 것은 흔하게 볼 수 있습니다. 
많은 언어들에서 이것은 `UselessJunk` 타입을 위한 공간을 할당하고, `UselessJunk`를 가지고 아무것도 하지 않기 위해서 그 값을 저장하고 읽는 작업을 강제할 겁니다. 
이 작업이 불필요하다는 것을 증명하려면 컴파일러는 복잡한 분석을 해야 할 겁니다.

그러나 러스트에서는 우리는 그냥 `Set<Key> = Map<Key, ()>`라고 말할 수 있습니다. 이제 러스트는 컴파일할 때 모든 메모리 읽기와 저장은 의미가 없고, 저장 공간을 할당할 필요도 없다는 것을 알게 됩니다. 
결과적으로 나오는 코드는 그냥 `HashSet`의 커스텀 구현일 뿐이고, `HashMap`이 값을 처리할 때의 연산은 존재하지 않게 됩니다.

안전한 코드는 무량 타입에 대해서 걱정하지 않아도 되지만, *불안전한* 코드는 크기가 없는 타입의 중요성을 신경써야 합니다. 특히 포인터 오프셋은 아무 작업도 하지 않는 것과 같고, 할당자는 보통 [0이 아닌 크기를 요구합니다][alloc].

무량 타입을 가리키는 레퍼런스(빈 슬라이스 포함)는 다른 레퍼런스와 마찬가지로, 널이 아니고 잘 정렬되어 있어야 합니다. 무량 타입을 가리키지만 널이나 정렬되지 않은 포인터를 역참조하는 것 역시, 다른 타입들과 마찬가지로 [미정의 동작][ub]입니다.

[alloc]: https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html#tymethod.alloc
[ub]: what-unsafe-does.html

## 빈 타입

러스트는 또한 *그 타입의 값을 만들 수조차 없는* 타입을 정의하는 것도 지원합니다. 이런 타입들은 타입 측면에서만 말할 수 있고, 값 측면에서는 절대 말할 수 없습니다. 
빈 타입은 형이 없는 열거형을 정의함으로써 만들 수 있습니다: 

```rust
enum Void {} // 형 없음 = 비어 있음
```
빈 타입은 무량 타입보다도 더 작습니다. 빈 타입의 예시로 들 만한 것은 타입 측면에서의 접근불가성입니다. 예를 들어, 어떤 API가 일반적으로 `Result`를 반환해야 하지만, 어떤 경우에서는 실패할 수 없다고 합시다. 
우리는 이것을 `Result<T, Void>`를 반환함으로써 타입 레벨에서 소통할 수 있습니다. 

Consumers of the API can confidently unwrap such a Result
knowing that it's *statically impossible* for this value to be an `Err`, as
this would require providing a value of type `Void`.

In principle, Rust can do some interesting analyses and optimizations based
on this fact. For instance, `Result<T, Void>` is represented as just `T`,
because the `Err` case doesn't actually exist (strictly speaking, this is only
an optimization that is not guaranteed, so for example transmuting one into the
other is still Undefined Behavior).

The following *could* also compile:

```rust,compile_fail
enum Void {}

let res: Result<u32, Void> = Ok(0);

// Err doesn't exist anymore, so Ok is actually irrefutable.
let Ok(num) = res;
```

But this trick doesn't work yet.

One final subtle detail about empty types is that raw pointers to them are
actually valid to construct, but dereferencing them is Undefined Behavior
because that wouldn't make sense.

We recommend against modelling C's `void*` type with `*const Void`.
A lot of people started doing that but quickly ran into trouble because
Rust doesn't really have any safety guards against trying to instantiate
empty types with unsafe code, and if you do it, it's Undefined Behavior.
This was especially problematic because developers had a habit of converting
raw pointers to references and `&Void` is *also* Undefined Behavior to
construct.

`*const ()` (or equivalent) works reasonably well for `void*`, and can be made
into a reference without any safety problems. It still doesn't prevent you from
trying to read or write values, but at least it compiles to a no-op instead
of Undefined Behavior.

## Extern Types

There is [an accepted RFC][extern-types] to add proper types with an unknown size,
called *extern types*, which would let Rust developers model things like C's `void*`
and other "declared but never defined" types more accurately. However as of
Rust 2018, [the feature is stuck in limbo over how `size_of_val::<MyExternType>()`
should behave][extern-types-issue].

[extern-types]: https://github.com/rust-lang/rfcs/blob/master/text/1861-extern-types.md
[extern-types-issue]: https://github.com/rust-lang/rust/issues/43467
[`str`]: https://doc.rust-lang.org/std/primitive.str.html
[slice]: https://doc.rust-lang.org/std/primitive.slice.html
