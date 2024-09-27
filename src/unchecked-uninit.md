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

3. 마지막으로, 우리는 배열의 타입에서 `MaybeUninit` 을 지워야 합니다. 현재의 안정적인 러스트 버전으로는, 이 작업은 `transmute`를 써야 합니다. 이 변질은 합당한데 이는 메모리 안에서 `MaybeUninit<T>`은 `T`와 똑같이 보이기 때문입니다.


    하지만, 보통은 `Container<MaybeUninit<T>>>`는 `Container<T>`와 똑같이 보이지 *않습니다*! 만약 `Container`가 `Option`이고, `T`가 `bool`이라고 가정할 때,
   `Option<bool>`은 `bool`이 오직 유효한 2개의 값을 가지고 있다는 것을 이용하지만, `Option<MaybeUninit<bool>>`은 `bool`이 초기화되지 않아도 되기 때문에 그런 작업을 할 수 없습니다.

    따라서, `MaybeUninit`을 변질해서 타입에서 지워도 되는지는 `Container`에 따라 다릅니다. 배열에 대해서는 그렇습니다 (그리고 결국 표준 라이브러리도 이것을 알아차리고 적당한 메서드를 제공할 겁니다).

중간에 있는 반복문은 좀더 시간을 들일 가치가 있는데, 특히 할당문과 또한 `drop` 간의 관계가 그렇습니다. 우리가 만약 이런 코드를 쓴다면:

<!-- ignore: simplified code -->
```rust,ignore
*x[i].as_mut_ptr() = Box::new(i as u32); // 틀림!
```

우리는 `Box<u32>`를 실제로 덮어쓰게 되고, 미초기화된 데이터가 `drop`되며, 이는 엄청난 슬픔과 고통으로 다가올 것입니다.



The correct alternative, if for some reason we cannot use `MaybeUninit::new`, is
to use the [`ptr`] module. In particular, it provides three functions that allow
us to assign bytes to a location in memory without dropping the old value:
[`write`], [`copy`], and [`copy_nonoverlapping`].

* `ptr::write(ptr, val)` takes a `val` and moves it into the address pointed
  to by `ptr`.
* `ptr::copy(src, dest, count)` copies the bits that `count` T items would occupy
  from src to dest. (this is equivalent to C's memmove -- note that the argument
  order is reversed!)
* `ptr::copy_nonoverlapping(src, dest, count)` does what `copy` does, but a
  little faster on the assumption that the two ranges of memory don't overlap.
  (this is equivalent to C's memcpy -- note that the argument order is reversed!)

It should go without saying that these functions, if misused, will cause serious
havoc or just straight up Undefined Behavior. The only requirement of these
functions *themselves* is that the locations you want to read and write
are allocated and properly aligned. However, the ways writing arbitrary bits to
arbitrary locations of memory can break things are basically uncountable!

It's worth noting that you don't need to worry about `ptr::write`-style
shenanigans with types which don't implement `Drop` or contain `Drop` types,
because Rust knows not to try to drop them. This is what we relied on in the
above example.

However when working with uninitialized memory you need to be ever-vigilant for
Rust trying to drop values you make like this before they're fully initialized.
Every control path through that variable's scope must initialize the value
before it ends, if it has a destructor.
*[This includes code panicking](unwinding.html)*. `MaybeUninit` helps a bit
here, because it does not implicitly drop its content - but all this really
means in case of a panic is that instead of a double-free of the not yet
initialized parts, you end up with a memory leak of the already initialized
parts.

Note that, to use the `ptr` methods, you need to first obtain a *raw pointer* to
the data you want to initialize. It is illegal to construct a *reference* to
uninitialized data, which implies that you have to be careful when obtaining
said raw pointer:

* For an array of `T`, you can use `base_ptr.add(idx)` where `base_ptr: *mut T`
to compute the address of array index `idx`. This relies on
how arrays are laid out in memory.
* For a struct, however, in general we do not know how it is laid out, and we
also cannot use `&mut base_ptr.field` as that would be creating a
reference. So, you must carefully use the [`addr_of_mut`] macro. This creates
a raw pointer to the field without creating an intermediate reference:

```rust
use std::{ptr, mem::MaybeUninit};

struct Demo {
    field: bool,
}

let mut uninit = MaybeUninit::<Demo>::uninit();
// `&uninit.as_mut().field` would create a reference to an uninitialized `bool`,
// and thus be Undefined Behavior!
let f1_ptr = unsafe { ptr::addr_of_mut!((*uninit.as_mut_ptr()).field) };
unsafe { f1_ptr.write(true); }

let init = unsafe { uninit.assume_init() };
```

One last remark: when reading old Rust code, you might stumble upon the
deprecated `mem::uninitialized` function.  That function used to be the only way
to deal with uninitialized memory on the stack, but it turned out to be
impossible to properly integrate with the rest of the language.  Always use
`MaybeUninit` instead in new code, and port old code over when you get the
opportunity.

And that's about it for working with uninitialized memory! Basically nothing
anywhere expects to be handed uninitialized memory, so if you're going to pass
it around at all, be sure to be *really* careful.

[`MaybeUninit`]: https://doc.rust-lang.org/core/mem/union.MaybeUninit.html
[assume_init]: https://doc.rust-lang.org/core/mem/union.MaybeUninit.html#method.assume_init
[`ptr`]: https://doc.rust-lang.org/core/ptr/index.html
[`addr_of_mut`]: https://doc.rust-lang.org/core/ptr/macro.addr_of_mut.html
[`write`]: https://doc.rust-lang.org/core/ptr/fn.write.html
[`copy`]: https://doc.rust-lang.org/std/ptr/fn.copy.html
[`copy_nonoverlapping`]: https://doc.rust-lang.org/std/ptr/fn.copy_nonoverlapping.html
