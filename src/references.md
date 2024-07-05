# 레퍼런스

레퍼런스에는 두 가지 종류가 있습니다:

* 불변 레퍼런스: `&`
* 가변 레퍼런스: `&mut`

이것들은 다음의 규칙들을 지킵니다:

* 레퍼런스는 그 본체보다 오래 살 수 없다
* 가변 레퍼런스는 복제할 수 없다

이게 다입니다. 이것이 레퍼런스가 따르는 모델 전부입니다.

물론, 우리는 *복제한다*는 것이 어떤 의미인지 정의해야겠죠.

```text
error[E0425]: cannot find value `aliased` in this scope
 --> <rust.rs>:2:20
  |
2 |     println!("{}", aliased);
  |                    ^^^^^^^ not found in this scope

error: aborting due to previous error
```

아쉽게도, 러스트는 아직 복제 모델을 정의하지 않았습니다. 🙀

러스트 개발자들이 언어의 의미를 확실히 정하는 동안, 다음 섹션에서 복제가 무엇인지, 왜 중요한지 논해 보겠습니다.
