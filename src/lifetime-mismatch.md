# 수명의 한계

다음 코드가 주어졌을 때:

```rust,compile_fail
#[derive(Debug)]
struct Foo;

impl Foo {
    fn mutate_and_share(&mut self) -> &Self { &*self }
    fn share(&self) {}
}

fn main() {
    let mut foo = Foo;
    let loan = foo.mutate_and_share();
    foo.share();
    println!("{:?}", loan);
}
```

이것이 컴파일이 되기를 기댜할 수도 있습니다. 우리는 `mutate_and_share`를 호출하는데, 이것은 `foo`를 잠시 가변으로 빌리지만, 불변 레퍼런스만 반환하기 때문입니다. 따라서 `foo`는 가변으로 빌린 상태가 아니므로 우리는 `foo.share()`가 성공할 것이라고 기대할 것입니다.

하지만 우리가 이것을 컴파일하려고 하면:

```text
error[E0502]: cannot borrow `foo` as immutable because it is also borrowed as mutable
  --> src/main.rs:12:5
   |
11 |     let loan = foo.mutate_and_share();
   |                --- mutable borrow occurs here
12 |     foo.share();
   |     ^^^ immutable borrow occurs here
13 |     println!("{:?}", loan);
```

무슨 일이 일어난 걸까요? 음, 이것은 [전 섹션의 2번째 예제][ex2]와 정확히 동일한 논법입니다. 우리가 이 프로그램을 해독하면 다음을 얻습니다:

<!-- ignore: desugared code -->
```rust,ignore
struct Foo;

impl Foo {
    fn mutate_and_share<'a>(&'a mut self) -> &'a Self { &'a *self }
    fn share<'a>(&'a self) {}
}

fn main() {
    'b: {
        let mut foo: Foo = Foo;
        'c: {
            let loan: &'c Foo = Foo::mutate_and_share::<'c>(&'c mut foo);
            'd: {
                Foo::share::<'d>(&'d foo);
            }
            println!("{:?}", loan);
        }
    }
}
```

수명 체계는 `&mut foo`의 수명을 `'c`로 늘려야 할 수밖에 없는데, 이는 `loan`의 수명과 `mutate_and_share`의 시그니처 때문입니다. 그 다음 우리가 `share`르ㄹ 호출하려 할 때, 수명 체계는 우리가 `&'c mut foo`를 복제하려는 것을 알아차리고 우리 눈 앞에서 터집니다!

이 프로그램은 우리가 실제로 신경쓰는 레퍼런스 의미론에 따르면 명확히 옳지만, 수명 체계는 그것을 전달하기에 너무 헐겁습니다.

## Improperly reduced borrows

The following code fails to compile, because Rust sees that a variable, `map`,
is borrowed twice, and can not infer that the first borrow ceases to be needed
before the second one occurs. This is caused by Rust conservatively falling back
to using a whole scope for the first borrow. This will eventually get fixed.

```rust,compile_fail
# use std::collections::HashMap;
# use std::hash::Hash;
fn get_default<'m, K, V>(map: &'m mut HashMap<K, V>, key: K) -> &'m mut V
where
    K: Clone + Eq + Hash,
    V: Default,
{
    match map.get_mut(&key) {
        Some(value) => value,
        None => {
            map.insert(key.clone(), V::default());
            map.get_mut(&key).unwrap()
        }
    }
}
```

Because of the lifetime restrictions imposed, `&mut map`'s lifetime
overlaps other mutable borrows, resulting in a compile error:

```text
error[E0499]: cannot borrow `*map` as mutable more than once at a time
  --> src/main.rs:12:13
   |
4  |   fn get_default<'m, K, V>(map: &'m mut HashMap<K, V>, key: K) -> &'m mut V
   |                  -- lifetime `'m` defined here
...
9  |       match map.get_mut(&key) {
   |       -     --- first mutable borrow occurs here
   |  _____|
   | |
10 | |         Some(value) => value,
11 | |         None => {
12 | |             map.insert(key.clone(), V::default());
   | |             ^^^ second mutable borrow occurs here
13 | |             map.get_mut(&key).unwrap()
14 | |         }
15 | |     }
   | |_____- returning this value requires that `*map` is borrowed for `'m`
```

[ex2]: lifetimes.html#example-aliasing-a-mutable-reference
