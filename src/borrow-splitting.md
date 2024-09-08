# 대여 쪼개기

가변 레퍼런스의 상호 배제 규칙은 복잡한 구조와 작업할 때 굉장히 제한적일 수 있습니다. 대여 검사기(즉 borrowck)는 기본적인 내용은 이해하지만, 금세 쉽게 넘어질 겁니다. 이것은 한 구조체의 다른 필드들을 빌리는 것이 가능하다는 
것은 알 정도로 구조체를 이해합니다. 따라서 이 코드는 오늘날 잘 작동합니다:

```rust
struct Foo {
    a: i32,
    b: i32,
    c: i32,
}

let mut x = Foo {a: 0, b: 0, c: 0};
let a = &mut x.a;
let b = &mut x.b;
let c = &x.c;
*b += 1;
let c2 = &x.c;
*a += 10;
println!("{} {} {} {}", a, b, c, c2);
```

하지만 borrowck는 배열이나 슬라이스는 전혀 이해하지 못해서, 이 코드는 동작하지 않습니다:

```rust,compile_fail
let mut x = [1, 2, 3];
let a = &mut x[0];
let b = &mut x[1];
println!("{} {}", a, b);
```

```text
error[E0499]: cannot borrow `x[..]` as mutable more than once at a time
 --> src/lib.rs:4:18
  |
3 |     let a = &mut x[0];
  |                  ---- first mutable borrow occurs here
4 |     let b = &mut x[1];
  |                  ^^^^ second mutable borrow occurs here
5 |     println!("{} {}", a, b);
6 | }
  | - first borrow ends here

error: aborting due to previous error
```

borrowck가 이런 간단한 경우를 이해할 수 있다는 것은 좋지만, borrowck가 트리 같은 일반적인 컨테이너 타입들에서 "다른 부분"이라는 개념을 이해할 가능성은 거의 없습니다, 특히 다른 키들이 같은 값에 *대응할* 때는요.

borrowck에게 우리가 하는 작업이 괜찮다는 것을 "가르쳐 주기" 위해, 우리는 불안전한 코드로 내려와야 합니다. 예를 들어 가변 슬라이스는 슬라이스를 소비하고 두 개의 가변 슬라이스를 반환하는 `split_at_mut` 함수를 정의합니다. 
하나는 인덱스의 왼쪽에 있는 모든 것의 슬라이스이고, 다른 하나는 인덱스의 오른쪽에 있는 모든 것의 슬라이스입니다. 직관적으로 우리는 이것이 안전하다는 것을 아는데, 이는 슬라이스가 서로 겹치지 않기 때문입니다. 하지만 이것을 구현하는 것은 
어느 정도의 불안전성을 요구합니다:

```rust
use std::slice::from_raw_parts_mut;
struct FakeSlice<T>(T);
impl<T> FakeSlice<T> {
    fn len(&self) -> usize { unimplemented!() }
    fn as_mut_ptr(&mut self) -> *mut T { unimplemented!() }
    pub fn split_at_mut(&mut self, mid: usize) -> (&mut [T], &mut [T]) {
        let len = self.len();
        let ptr = self.as_mut_ptr();

        unsafe {
            assert!(mid <= len);

            (from_raw_parts_mut(ptr, mid),
             from_raw_parts_mut(ptr.add(mid), len - mid))
        }
    }
}
```

이것은 사실 조금 애매합니다. 같은 값에 두 개의 `&mut`을 만드는 일이 없도록, 우리는 생 포인터를 통하여 명시적으로 새로운 슬라이스를 만듭니다.

그러나 더욱 애매한 것은 가변 레퍼런스를 반환하는 반복자들이 어떻게 작동하느냐 하는 것입니다. `Iterator` 트레잇은 다음과 같이 정의되어 있습니다:

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

이 정의에 의하면, `Self::Item`은 `self`와 어떤 관련도 *없습니다*. 이것이 의미하는 것은 우리가 `next`를 여러 번 연속으로 호출한 후, 결과들을 *동시에* 기다리는 것이 가능하다는 겁니다. 
이것은 값을 넘겨주는 반복자들에게는 아무 영향도 없습니다, 정확히 이런 의미를 가지거든요. 불변 레퍼런스를 반환하는 반복자들에게도 문제 없습니다, 같은 값에 임의의 많은 레퍼런스들이 있는 것을 허용하기 때문이죠 
(비록 반복자가 공유되는 값과 다른 객체여야 하지만요).

하지만 가변 레퍼런스들이 이것을 망칩니다. 처음에 보기에는, 이 API와 전혀 호환될 것 같지 않은데, 이는 같은 객체에 여러 개의 가변 레퍼런스를 만들 것이기 때문입니다!

그러나 실제로는 잘 *동작하는데*, 바로 반복자들이 일회용 객체들이기 때문입니다. `IterMut`이 반환하는 모든 것은 최대 1번 반환될 것이므로, 우리는 같은 데이터 조각에 여러 개의 가변 레퍼런스를 반환하는 일은 없을 겁니다.

놀라울지는 모르겠지만, 가변 반복자들은 많은 타입들의 경우에 구현하는 데에 불안전한 코드가 필요하지 않습니다!

예를 들어 단일 링크드 리스트입니다:

```rust
# fn main() {}
type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

pub struct LinkedList<T> {
    head: Link<T>,
}

pub struct IterMut<'a, T: 'a>(Option<&'a mut Node<T>>);

impl<T> LinkedList<T> {
    fn iter_mut(&mut self) -> IterMut<T> {
        IterMut(self.head.as_mut().map(|node| &mut **node))
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node| {
            self.0 = node.next.as_mut().map(|node| &mut **node);
            &mut node.elem
        })
    }
}
```

가변 슬라이스입니다:

```rust
# fn main() {}
use std::mem;

pub struct IterMut<'a, T: 'a>(&'a mut[T]);

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        let slice = mem::take(&mut self.0);
        if slice.is_empty() { return None; }

        let (l, r) = slice.split_at_mut(1);
        self.0 = r;
        l.get_mut(0)
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        let slice = mem::take(&mut self.0);
        if slice.is_empty() { return None; }

        let new_len = slice.len() - 1;
        let (l, r) = slice.split_at_mut(new_len);
        self.0 = l;
        r.get_mut(0)
    }
}
```

그리고 이진 트리입니다:

```rust
# fn main() {}
use std::collections::VecDeque;

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    left: Link<T>,
    right: Link<T>,
}

pub struct Tree<T> {
    root: Link<T>,
}

struct NodeIterMut<'a, T: 'a> {
    elem: Option<&'a mut T>,
    left: Option<&'a mut Node<T>>,
    right: Option<&'a mut Node<T>>,
}

enum State<'a, T: 'a> {
    Elem(&'a mut T),
    Node(&'a mut Node<T>),
}

pub struct IterMut<'a, T: 'a>(VecDeque<NodeIterMut<'a, T>>);

impl<T> Tree<T> {
    pub fn iter_mut(&mut self) -> IterMut<T> {
        let mut deque = VecDeque::new();
        self.root.as_mut().map(|root| deque.push_front(root.iter_mut()));
        IterMut(deque)
    }
}

impl<T> Node<T> {
    pub fn iter_mut(&mut self) -> NodeIterMut<T> {
        NodeIterMut {
            elem: Some(&mut self.elem),
            left: self.left.as_mut().map(|node| &mut **node),
            right: self.right.as_mut().map(|node| &mut **node),
        }
    }
}


impl<'a, T> Iterator for NodeIterMut<'a, T> {
    type Item = State<'a, T>;

    fn next(&mut self) -> Option<Self::Item> {
        match self.left.take() {
            Some(node) => Some(State::Node(node)),
            None => match self.elem.take() {
                Some(elem) => Some(State::Elem(elem)),
                None => match self.right.take() {
                    Some(node) => Some(State::Node(node)),
                    None => None,
                }
            }
        }
    }
}

impl<'a, T> DoubleEndedIterator for NodeIterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        match self.right.take() {
            Some(node) => Some(State::Node(node)),
            None => match self.elem.take() {
                Some(elem) => Some(State::Elem(elem)),
                None => match self.left.take() {
                    Some(node) => Some(State::Node(node)),
                    None => None,
                }
            }
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        loop {
            match self.0.front_mut().and_then(|node_it| node_it.next()) {
                Some(State::Elem(elem)) => return Some(elem),
                Some(State::Node(node)) => self.0.push_front(node.iter_mut()),
                None => if let None = self.0.pop_front() { return None },
            }
        }
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        loop {
            match self.0.back_mut().and_then(|node_it| node_it.next_back()) {
                Some(State::Elem(elem)) => return Some(elem),
                Some(State::Node(node)) => self.0.push_back(node.iter_mut()),
                None => if let None = self.0.pop_back() { return None },
            }
        }
    }
}
```

All of these are completely safe and work on stable Rust! This ultimately
falls out of the simple struct case we saw before: Rust understands that you
can safely split a mutable reference into subfields. We can then encode
permanently consuming a reference via Options (or in the case of slices,
replacing with an empty slice).
