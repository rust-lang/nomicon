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

    // prints: "17 [0, 0, 0, 0, 0, 0, 0, 0]"
    println!("{} {:?}", dynamic.info, &dynamic.data);
}
```

(네, 커스텀 동량 타입은 지긍으로써는 매우 설익은 기능입니다.)

## 무량(無量) 타입 (ZST)

Rust also allows types to be specified that occupy no space:

```rust
struct Nothing; // No fields = no size

// All fields have no size = no size
struct LotsOfNothing {
    foo: Nothing,
    qux: (),      // empty tuple has no size
    baz: [u8; 0], // empty array has no size
}
```

On their own, Zero Sized Types (ZSTs) are, for obvious reasons, pretty useless.
However as with many curious layout choices in Rust, their potential is realized
in a generic context: Rust largely understands that any operation that produces
or stores a ZST can be reduced to a no-op. First off, storing it doesn't even
make sense -- it doesn't occupy any space. Also there's only one value of that
type, so anything that loads it can just produce it from the aether -- which is
also a no-op since it doesn't occupy any space.

One of the most extreme examples of this is Sets and Maps. Given a
`Map<Key, Value>`, it is common to implement a `Set<Key>` as just a thin wrapper
around `Map<Key, UselessJunk>`. In many languages, this would necessitate
allocating space for UselessJunk and doing work to store and load UselessJunk
only to discard it. Proving this unnecessary would be a difficult analysis for
the compiler.

However in Rust, we can just say that  `Set<Key> = Map<Key, ()>`. Now Rust
statically knows that every load and store is useless, and no allocation has any
size. The result is that the monomorphized code is basically a custom
implementation of a HashSet with none of the overhead that HashMap would have to
support values.

Safe code need not worry about ZSTs, but *unsafe* code must be careful about the
consequence of types with no size. In particular, pointer offsets are no-ops,
and allocators typically [require a non-zero size][alloc].

Note that references to ZSTs (including empty slices), just like all other
references, must be non-null and suitably aligned. Dereferencing a null or
unaligned pointer to a ZST is [undefined behavior][ub], just like for any other
type.

[alloc]: https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html#tymethod.alloc
[ub]: what-unsafe-does.html

## Empty Types

Rust also enables types to be declared that *cannot even be instantiated*. These
types can only be talked about at the type level, and never at the value level.
Empty types can be declared by specifying an enum with no variants:

```rust
enum Void {} // No variants = EMPTY
```

Empty types are even more marginal than ZSTs. The primary motivating example for
an empty type is type-level unreachability. For instance, suppose an API needs to
return a Result in general, but a specific case actually is infallible. It's
actually possible to communicate this at the type level by returning a
`Result<T, Void>`. Consumers of the API can confidently unwrap such a Result
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
