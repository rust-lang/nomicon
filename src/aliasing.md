# 복제

먼저, 몇 가지 짚고 넘어가겠습니다:

* 논의를 위해 가능한 한 가장 넓은 복제의 정의를 사용할 것입니다. 러스트의 정의는 아마 변형이나 살아있음 여부에 대해서는 조금 더 제한적일 것입니다.

* 우리는 한 스레드에서, 인터럽트 없는 실행을 가정하겠습니다. 또한 메모리-매핑된 하드웨어 같은 것들은 무시하겠습니다. 러스트는 당신이 말하지 않는 한 이런 것들이 일어나지 않는다고 가정합니다. 더 자세한 내용은 [동시성 챕터를](concurrency.html) 참고하세요.

이런 것들을 말했으니, 이제 우리의 아직 작업중인 정의를 말해보겠습니다: 변수들과 포인터들은 메모리의 겹쳐지는 지역을 가리킬 때 *복제되었다*고 합니다.

## 복제가 중요한 이유

그래서 왜 우리가 복제를 신경써야 할까요?

이런 간단한 함수를 생각해 보세요:

```rust
fn compute(input: &u32, output: &mut u32) {
    if *input > 10 {
        *output = 1;
    }
    if *input > 5 {
        *output *= 2;
    }
    // `input > 10`일 경우 `output`은 `2`가 될 것이라는 것을 기억하세요
}
```

이 함수를 다음의 함수로 최적화할 수 있다면 *좋겠네요*:

```rust
fn compute(input: &u32, output: &mut u32) {
    let cached_input = *input; // `*input`을 레지스터에 저장합니다.
    if cached_input > 10 {
        // `input`이 10보다 크면, 이전 코드는 `output`을 1로 지정했다가 2를 곱하니, 
        // `output`은 최종적으로 2가 됩니다 (`>10`이면 `>5`일 테니까요).
        // 여기서는 두번 할당하는 것을 피하고 한번에 2로 설정합니다.
        *output = 2;
    } else if cached_input > 5 {
        *output *= 2;
    }
}
```

러스트에서는 이런 최적화가 건전할 것입니다. 거의 모든 다른 언어에서는 그렇지 않을 것입니다 (전역 분석을 제외하면). 이것은 이 최적화가 복제가 일어나지 않는다는 것에 의존하기 때문인데, 많은 언어들이 이것에 있어서 자유롭게 풀어두죠. 
특별히 우리는 `input`과 `output`이 겹치는 함수 매개변수들, 예를 들면 `compute(&x, &mut x)` 같은 것들을 걱정해야 합니다.

이런 입력으로는 이런 실행이 가능합니다:

<!-- ignore: expanded code -->
```rust,ignore
                    //  input ==  output == 0xabad1dea
                    // *input == *output == 20
if *input > 10 {    // 참  (*input == 20)
    *output = 1;    // *input 에도 씀, *output 과 같기 때문
}
if *input > 5 {     // 거짓 (*input == 1)
    *output *= 2;
}
                    // *input == *output == 1
```

우리의 최적화된 함수는 이런 입력에 `*output == 2`라는 결과를 도출할 것이고, 따라서 우리의 최적화가 올바른지의 문제는 이런 입력이 불가능하다는 것에 기반합니다.

우리는 러스트에서는 `&mut`를 복제하는 것이 허락되지 않기 때문에, 이런 입력이 불가능하다는 것을 압니다. 따라서 우리는 안전하게 그런 가능성을 거부하고 최적화를 실행할 수 있게 됩니다. 
다른 대부분의 언어들에서는 이런 입력 또한 완전히 가능할 것이고, 고려되어야 할 것입니다.

이것이 바로 복제 분석이 중요한 이유입니다: 유용한 최적화를 컴파일러가 실행하도록 해 주거든요! 몇 가지 예를 들자면:

* 값의 메모리를 접근하는 포인터가 없다는 것을 증명함으로써 값들을 레지스터에 그대로 두는 것
* 어떤 메모리는 마지막으로 읽은 후에 쓴 적이 없다는 것을 증명함으로써 읽기 작업들을 제거하는 것
* 어떤 메모리는 다음 쓰기 작업 전에 읽은 적이 없다는 것을 증명함으로써 쓰기 작업들을 제거하는 것
* 읽기 작업들이나 쓰기 작업들이 서로에 의존하지 않는다는 것을 증명함으로써 작업들을 옭기거나 순서를 바꾸는 것

이런 최적화는 또한 루프 벡터화, 상수 전파, 죽은 코드 제거 등의 더 큰 최적화의 건전함을 증명하게 되는 경향이 있습니다.

이전의 예제에서, 우리는 `&mut u32`가 복제될 수 없다는 사실을 이용해서 `*output`에 쓰는 작업이 `*input`에 영향을 줄 수 없다는 것을 증명했습니다. 이러면 우리는 레지스터에 `*input`을 캐싱해서, 읽기 작업을 제거할 수 있습니다.

이 읽기 작업을 캐싱함으로써, 우리는 `> 10` 분기에서 있는 쓰기 작업이 `> 5` 분기를 택하는지 여부를 영향주지 못한다는 것을 알게 되고, `*input > 10`일 때에 읽고, 수정하고, 다시 쓰는 작업(`*output`을 2배로 하는 작업)을 제거할 수 있게 됩니다.



The key thing to remember about alias analysis is that writes are the primary
hazard for optimizations. That is, the only thing that prevents us
from moving a read to any other part of the program is the possibility of us
re-ordering it with a write to the same location.

For instance, we have no concern for aliasing in the following modified version
of our function, because we've moved the only write to `*output` to the very
end of our function. This allows us to freely reorder the reads of `*input` that
occur before it:

```rust
fn compute(input: &u32, output: &mut u32) {
    let mut temp = *output;
    if *input > 10 {
        temp = 1;
    }
    if *input > 5 {
        temp *= 2;
    }
    *output = temp;
}
```

We're still relying on alias analysis to assume that `input` doesn't alias
`temp`, but the proof is much simpler: the value of a local variable can't be
aliased by things that existed before it was declared. This is an assumption
every language freely makes, and so this version of the function could be
optimized the way we want in any language.

This is why the definition of "alias" that Rust will use likely involves some
notion of liveness and mutation: we don't actually care if aliasing occurs if
there aren't any actual writes to memory happening.

Of course, a full aliasing model for Rust must also take into consideration things like
function calls (which may mutate things we don't see), raw pointers (which have
no aliasing requirements on their own), and UnsafeCell (which lets the referent
of an `&` be mutated).
