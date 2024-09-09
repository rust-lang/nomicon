# 강제 변환

타입들은 특정한 상황들에서는 묵시적으로 강제 변환될 수 있습니다. 이런 변화들은 일반적으로 타입 시스템을 *약화시키는* 것인데, 주로 포인터와 수명에 초점이 맞춰져 있습니다. 이런 것들은 주로 많은 상황에서 러스트가 "그냥 잘 작동"하도록 하기 위하여 존재하며, 꽤나 위험성이 없습니다.

모든 강제 변환의 종류에 대해서는 참조서의 [강제 변환 타입][coercion-types] 섹션을 보세요.

트레잇을 매칭할 때는 강제 변환을 실행하지 않는다는 것을 유의하세요 (except for receivers, [다음 페이지][dot-operator]를 보세요). 만약 어떤 타입 `U`를 위한 `impl`이 있고 `T`가 `U`로 강제 변환된다면, `T`를 위한 구현으로 인정되지는 않습니다. 
예를 들어 다음의 코드는 타입 검사를 통과하지 못할 텐데, `t`를 `&T`로 강제 변환해도 괜찮고 `&T`를 위한 `impl`이 있는데도 그렇습니다:

```rust,compile_fail
trait Trait {}

fn foo<X: Trait>(t: X) {}

impl<'a> Trait for &'a i32 {}

fn main() {
    let t: &mut i32 = &mut 0;
    foo(t);
}
```

이는 다음의 에러를 내뱉습니다:

```text
error[E0277]: the trait bound `&mut i32: Trait` is not satisfied
 --> src/main.rs:9:9
  |
3 | fn foo<X: Trait>(t: X) {}
  |           ----- required by this bound in `foo`
...
9 |     foo(t);
  |         ^ the trait `Trait` is not implemented for `&mut i32`
  |
  = help: the following implementations were found:
            <&'a i32 as Trait>
  = note: `Trait` is implemented for `&i32`, but not for `&mut i32`
```

[coercion-types]: https://doc.rust-lang.org/reference/type-coercions.html#coercion-types
[dot-operator]: ./dot-operator.html
