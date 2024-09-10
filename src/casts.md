# 변형

변형은 강제 변환을 포함하는 더 큰 개념입니다: 모든 강제 변환은 변형으로 명시적으로 호출할 수 있습니다. 하지만 어떤 변환들은 변형을 필요로 합니다. 강제 변환은 흔하고 보통은 위험하지 않지만, 이런 "진짜 변형"은 희귀하고, 위험할 수 있습니다. 
그런 면에서, 변형은 명시적으로 `as` 키워드를 사용해서 호출해야 합니다: `expr as Type`.



You can find an exhaustive list of [all the true casts][cast_list] and [casting semantics][semantics_list] on the reference.

## Safety of casting

True casts generally revolve around raw pointers and the primitive numeric types.
Even though they're dangerous, these casts are infallible at runtime.
If a cast triggers some subtle corner case no indication will be given that this occurred.
The cast will simply succeed.
That said, casts must be valid at the type level, or else they will be prevented statically.
For instance, `7u8 as bool` will not compile.

That said, casts aren't `unsafe` because they generally can't violate memory safety *on their own*.
For instance, converting an integer to a raw pointer can very easily lead to terrible things.
However the act of creating the pointer itself is safe, because actually using a raw pointer is already marked as `unsafe`.

## Some notes about casting

### Lengths when casting raw slices

Note that lengths are not adjusted when casting raw slices; `*const [u16] as *const [u8]` creates a slice that only includes half of the original memory.

### Transitivity

Casting is not transitive, that is, even if `e as U1 as U2` is a valid expression, `e as U2` is not necessarily so.

[cast_list]: https://doc.rust-lang.org/reference/expressions/operator-expr.html#type-cast-expressions
[semantics_list]: https://doc.rust-lang.org/reference/expressions/operator-expr.html#semantics
