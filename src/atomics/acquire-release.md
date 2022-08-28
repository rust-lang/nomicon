# Acquire and Release

Next, we’re going to try and implement one of the simplest concurrent utilities
possible — a mutex, but without support for waiting (since that’s not really
related to what we’re doing now). It will hold both an atomic flag that
indicates whether it is locked or not, and the protected data itself. In code
this translates to:

```rs
use std::cell::UnsafeCell;
use std::sync::atomic::AtomicBool;

pub struct Mutex<T> {
    locked: AtomicBool,
    data: UnsafeCell<T>,
}

impl<T> Mutex<T> {
    pub const fn new(data: T) -> Self {
        Self {
            locked: AtomicBool::new(false),
            data: UnsafeCell::new(data),
        }
    }
}
```

Now for the lock function. We need to use an RMW here, since we need to both
check whether it is locked and lock it if is not in a single atomic step; this
can be most simply done with a `compare_exchange` (unlike before, it doesn’t
need to be in a loop this time). For the ordering, we’ll just use `Relaxed`
since we don’t know of any others yet.

```rust
# use std::cell::UnsafeCell;
# use std::sync::atomic::{self, AtomicBool};
# pub struct Mutex<T> {
#     locked: AtomicBool,
#     data: UnsafeCell<T>,
# }
impl<T> Mutex<T> {
    pub fn lock(&self) -> Option<Guard<'_, T>> {
        match self.locked.compare_exchange(
            false,
            true,
            atomic::Ordering::Relaxed,
            atomic::Ordering::Relaxed,
        ) {
            Ok(_) => Some(Guard(self)),
            Err(_) => None,
        }
    }
}

pub struct Guard<'mutex, T>(&'mutex Mutex<T>);
// Deref impl omitted…
```

We also need to implement `Drop` for `Guard` to make sure the lock on the mutex
is released once the guard is destroyed. Again we’re just using the `Relaxed`
ordering.

```rust
# use std::cell::UnsafeCell;
# use std::sync::atomic::{self, AtomicBool};
# pub struct Mutex<T> {
#     locked: AtomicBool,
#     data: UnsafeCell<T>,
# }
# pub struct Guard<'mutex, T>(&'mutex Mutex<T>);
impl<T> Drop for Guard<'_, T> {
    fn drop(&mut self) {
        self.0.locked.store(false, atomic::Ordering::Relaxed);
    }
}
```

Great! In the normal operation then, this primitive should allow unique access
to the data of the mutex to be transferred across different threads. Usual usage
could look like this:

```rust,ignore
// Initial state
let mutex = Mutex::new(0);
// Thread 1
if let Some(guard) = mutex.lock() {
    *guard += 1;
}
// Thread 2
if let Some(guard) = mutex.lock() {
    println!("{}", *guard);
}
```

Now, there are many possible executions of this code. For example, Thread 2 (the
reader thread) could lock the mutex first, and Thread 1 (the writer thread)
could fail to lock it:

```text
Thread 1      locked    data      Thread 2
╭───────╮   ┌────────┐ ┌───┐     ╭───────╮
│  cas  ├─┐ │ false  │ │ 0 ├╌┐ ┌─┤  cas  │
╰───────╯ │ └────────┘ └───┘ ┊ │ ╰───╥───╯
          │ ┌────────┬───────┼─┘ ╭───⇓───╮
          └─┤  true  │       └╌╌╌┤ guard │
            └────────┘           ╰───╥───╯
            ┌────────┬─────────┐ ╭───⇓───╮
            │ false  │         └─┤ store │
            └────────┘           ╰───────╯
```

Or potentially Thread _1_ could lock the mutex first, and Thread _2_ could fail
to lock it:

```text
Thread 1      locked      data    Thread 2
╭───────╮   ┌────────┐   ┌───┐   ╭───────╮
│  cas  ├─┐ │ false  │ ┌─│ 0 │───┤  cas  │
╰───╥───╯ │ └────────┘ │┌┼╌╌╌┤   ╰───────╯
╭───⇓───╮ └─┬────────┐ │├┼╌╌╌┤
│ += 1; ├╌┐ │  true  ├─┘┊│ 1 │
╰───╥───╯ ┊ └────────┘  ┊└───┘
╭───⇓───╮ └╌╌╌╌╌╌╌╌╌╌╌╌╌┘
│ store ├───┬────────┐
╰───────╯   │ false  │
            └────────┘
```

But the interesting case comes in when Thread 1 successfully locks and unlocks
the mutex, and then Thread 2 locks it. Let’s draw that one out too:

```text
Thread 1      locked     data       Thread 2
╭───────╮   ┌────────┐   ┌───┐     ╭───────╮
│  cas  ├─┐ │ false  │   │ 0 │ ┌───┤  cas  │
╰───╥───╯ │ └────────┘  ┌┼╌╌╌┤ │   ╰───╥───╯
╭───⇓───╮ └─┬────────┐  ├┼╌╌╌┤ │   ╭───⇓───╮
│ += 1; ├╌┐ │  true  │  ┊│ 1 │ │ ?╌┤ guard │
╰───╥───╯ ┊ └────────┘  ┊└───┘ │   ╰───╥───╯
╭───⇓───╮ └╌╌╌╌╌╌╌╌╌╌╌╌╌┘      │   ╭───⇓───╮
│ store ├───┬────────┐         │ ┌─┤ store │
╰───────╯   │ false  │         │ │ ╰───────╯
            └────────┘         │ │
            ┌────────┬─────────┘ │
            │  true  │           │
            └────────┘           │
            ┌────────┬───────────┘
            │ false  │
            └────────┘
```

Look at the second operation Thread 2 performs (the read of `data`), for which
we haven’t yet joined the line. Where should it connect to? Well actually, it
has multiple options…wait, we’ve seen this before! It’s a data race!

That’s not good. Last time the solution was to use atomics instead — but in this
case that doesn’t seem to be enough, since even if atomics were used it still
would have the _option_ of reading `0` instead of `1`, and really if we want our
mutex to be sane, it should only be able to read `1`.

So it seems that want we _want_ is to be able to apply the coherence rules from
before to completely rule out zero from the set of the possible values — if we
were able to draw a large arrow from the Thread 1’s `+= 1;` to Thread 2’s
`guard`, then we could trivially then use the rule to rule out `0` as a value
that could be read.

This is where the `Acquire` and `Release` orderings come in. Informally put, a
_release store_ will cause an arrow instead of a line to be drawn from the
operation to the destination; and similarly an _acquire load_ will cause an
arrow to be drawn from the destination to the operation. To give a useless
example that illustrates this, for the given program:

```rust
# use std::sync::atomic::{self, AtomicU32};
// Initial state
let a = AtomicU32::new(0);
// Thread 1
a.store(1, atomic::Ordering::Release);
// Thread 2
a.load(atomic::Ordering::Acquire);
```

The two possible executions look like this:

```text
    Possible Execution 1      ┃      Possible Execution 2
                              ┃
Thread 1      a     Thread 2  ┃  Thread 1      a     Thread 2
╭───────╮   ┌───┐   ╭──────╮  ┃  ╭───────╮   ┌───┐   ╭──────╮
│ store ├─┐ │ 0 │ ┌─→ load │  ┃  │ store ├─┐ │ 0 ├───→ load │
╰───────╯ │ └───┘ │ ╰──────╯  ┃  ╰───────╯ │ └───┘   ╰──────╯
          └─↘───┐ │           ┃            └─↘───┐
            │ 1 ├─┘           ┃              │ 1 │
            └───┘             ┃              └───┘
```

These arrows are a new kind of arrow we haven’t seen yet; they are known as
_happens-before_ (or happens-after) relations and are represented as thin arrows
(→) on these diagrams. They are weaker than the _sequenced before_
double-arrows (⇒) that occur inside a single thread, but can still be used with
the coherence rules to determine which values of a memory location are valid to
read.

When a happens-before arrow stores a data value to an atomic (via a release
operation) which is then loaded by another happens-before arrow (via an acquire
operation) we say that the release operation _synchronized-with_ the acquire
operation, which in doing so establishes that the release operation
_happens-before_ the acquire operation. Therefore, we can say that in the first
possible execution, Thread 1’s `store` synchronizes-with Thread 2’s `load`,
which causes that `store` and everything sequenced before it to happen-before
the `load` and everything sequenced after it.

> More formally, we can say that A happens-before B if any of the following
> conditions are true:
> 1. A is sequenced-before B (i.e. A occurs before B on the same thread)
> 2. A synchronizes-with B (i.e. A is a `Release` operation and B is an
>    `Acquire` operation that reads the value written by A)
> 3. A happens-before X, and X happens-before B (transitivity)

There is one more rule required for these to be useful, and that is _release
sequences_: after a release store is performed on an atomic, happens-before
arrows will connect together each subsequent value of the atomic as long as the
new value is caused by an RMW and not just a plain store (this means any
subsequent normal store, no matter the ordering, will end the sequence).

> In the C++11 memory model, any subsequent store by the same thread that
> performed the original `Release` store would also contribute to the release
> sequence. However, this was removed in C++20 for simplicity and better
> optimizations and so **must not** be relied upon.

With those rules in mind, converting Thread 1’s second store to use a `Release`
ordering as well as converting Thread 2’s CAS to use an `Acquire` ordering
allows us to effectively draw that arrow we needed before:

```text
Thread 1     locked     data       Thread 2
╭───────╮   ┌───────┐   ┌───┐     ╭───────╮
│  cas  ├─┐ │ false │   │ 0 │ ┌───→  cas  │
╰───╥───╯ │ └───────┘  ┌┼╌╌╌┤ │   ╰───╥───╯
╭───⇓───╮ └─┬───────┐  ├┼╌╌╌┤ │   ╭───⇓───╮
│ += 1; ├╌┐ │ true  │  ┊│ 1 ├╌│╌╌╌┤ guard │
╰───╥───╯ ┊ └───────┘  ┊└───┘ │   ╰───╥───╯
╭───⇓───╮ └╌╌╌╌╌╌╌╌╌╌╌╌┘      │   ╭───⇓───╮
│ store ├───↘───────┐         │ ┌─┤ store │
╰───────╯   │ false │         │ │ ╰───────╯
            └───┬───┘         │ │
            ┌───↓───┬─────────┘ │
            │ true  │           │
            └───────┘           │
            ┌───────┬───────────┘
            │ false │
            └───────┘
```

We now can trace back along the reverse direction of arrows from the `guard`
bubble to the `+= 1` bubble; we have established that Thread 2’s load
happens-after the `+= 1` side effect, because Thread 2’s CAS synchronizes-with
Thread 1’s store. This both avoids the data race and gives the guarantee that
`1` will be always read by Thread 2 (as long as locks after Thread 1, of
course).

However, that is not the only execution of the program possible. Even with this
setup, there is another execution that can also cause UB: if Thread 2 locks the
mutex before Thread 1 does.

```text
Thread 1       locked     data      Thread 2
╭───────╮     ┌───────┐   ┌───┐    ╭───────╮
│  cas  ├───┐ │ false │┌──│ 0 │────→  cas  │
╰───╥───╯   │ └───────┘│ ┌┼╌╌╌┤    ╰───╥───╯
╭───⇓───╮   │ ┌───────┬┘ ├┼╌╌╌┤    ╭───⇓───╮
│ += 1; ├╌┐ │ │ true  │  ┊│ 1 │  ?╌┤ guard │
╰───╥───╯ ┊ │ └───────┘  ┊└───┘    ╰───╥───╯
╭───⇓───╮ └╌│╌╌╌╌╌╌╌╌╌╌╌╌┘         ╭───⇓───╮
│ store ├─┐ │ ┌───────┬────────────┤ store │
╰───────╯ │ │ │ false │            ╰───────╯
          │ │ └───────┘
          │ └─┬───────┐
          │   │ true  │
          │   └───────┘
          └───↘───────┐
              │ false │
              └───────┘
```

Once again `guard` has multiple options for values to read. This one’s a bit
more counterintuitive than the previous one, since it requires “travelling
forward in time” to understand why the `1` is even there in the first place —
but since the abstract machine has no concept of time, it’s just a valid UB as
any other.

Luckily, we’ve already solved this problem once, so it easy to solve again: just
like before, we’ll have the CAS become acquire and the store become release, and
then we can use the second coherence rule from before to follow _forward_ the
arrow from the `guard` bubble all the way to the `+= 1;`, determining that it is
only possible for that read to see `0` as its value, as in the execution below.

```text
Thread 1       locked     data      Thread 2
╭───────╮     ┌───────┐   ┌───┐    ╭───────╮
│  cas  ←───┐ │ false │┌──│ 0 ├╌┐──→  cas  │
╰───╥───╯   │ └───────┘│ ┌┼╌╌╌┤ ┊  ╰───╥───╯
╭───⇓───╮   │ ┌───────┬┘ ├┼╌╌╌┤ ┊  ╭───⇓───╮
│ += 1; ├╌┐ │ │ true  │  ┊│ 1 │ └─╌┤ guard │
╰───╥───╯ ┊ │ └───────┘  ┊└───┘    ╰───╥───╯
╭───⇓───╮ └╌│╌╌╌╌╌╌╌╌╌╌╌╌┘         ╭───⇓───╮
│ store ├─┐ │ ┌───────↙────────────┤ store │
╰───────╯ │ │ │ false │            ╰───────╯
          │ │ └───┬───┘
          │ └─┬───↓───┐
          │   │ true  │
          │   └───────┘
          └───↘───────┐
              │ false │
              └───────┘
```

This leads us to the proper memory orderings for any mutex (and other locks like
RW locks too, even): use `Acquire` to lock it, and `Release` to unlock it. So
let’s go back to and update our original mutex definition with this knowledge.

But wait, `compare_exchange` takes two ordering parameters, not just one! That’s
right — it also takes a second one to apply when the exchange fails (in our case,
when the mutex is already locked). But we don’t need an `Acquire` here, since in
that case we won’t be reading from the `data` value anyway, so we’ll just stick
with `Relaxed`.

```rust,ignore
impl<T> Mutex<T> {
    pub fn lock(&self) -> Option<Guard<'_, T>> {
        match self.locked.compare_exchange(
            false,
            true,
            atomic::Ordering::Acquire,
            atomic::Ordering::Relaxed,
        ) {
            Ok(_) => Some(Guard(self)),
            Err(_) => None,
        }
    }
}

impl<T> Drop for Guard<'_, T> {
    fn drop(&mut self) {
        self.0.locked.store(false, atomic::Ordering::Release);
    }
}
```

Note that similarly to how atomic operations only make sense when paired with
other atomic operations on the same locations, `Acquire` only makes sense when
paired with `Release` and vice versa. That is, both an `Acquire` with no
corresponding `Release` and a `Release` with no corresponding `Acquire` are
useless, since the arrows will be unable to connect to anything.
