# Fences

As well as loads, stores, and RMWs, there is one more kind of atomic operation
to be aware of: fences. Fences can be triggered by the
[`core::sync::atomic::fence`] function, which accepts a single ordering
parameter and returns nothing. They don’t do anything on their own, but can be
thought of as events that strengthen the ordering of nearby atomic operations.

## Acquire fences

The most common kind of fence is an _acquire fence_, which can be triggered in
three different ways:
1. `atomic::fence(atomic::Ordering::Acquire)`
1. `atomic::fence(atomic::Ordering::AcqRel)`
1. `atomic::fence(atomic::Ordering::SeqCst)`

An acquire fence retroactively makes every single non-`Acquire` operation that
was sequenced-before it act like an `Acquire` operation that occurred at the
fence — in other words, it causes every prior `Release`d value that was
previously loaded on the thread to synchronize-with the fence. For example, the
following code:

```rust
# use std::sync::atomic::{self, AtomicU32};
static X: AtomicU32 = AtomicU32::new(0);

// t_1
X.store(1, atomic::Ordering::Release);

// t_2
let value = X.load(atomic::Ordering::Relaxed);
atomic::fence(atomic::Ordering::Acquire);
```

Can result in two possible executions:

```text
      Possible Execution 1      ┃      Possible Execution 2
                                ┃
   t_1        X        t_2      ┃      t_1        X        t_2
╭───────╮   ┌───┐   ╭───────╮   ┃   ╭───────╮   ┌───┐   ╭───────╮
│ store ├─┐ │ 0 │ ┌─┤ load  │   ┃   │ store ├─┐ │ 0 ├───┤ load  │
╰───────╯ │ └───┘ │ ╰───╥───╯   ┃   ╰───────╯ │ └───┘   ╰───╥───╯
          └─↘───┐ │ ╭───⇓───╮   ┃             └─↘───┐   ╭───⇓───╮
            │ 1 ├─┘┌→ fence │   ┃               │ 1 │   │ fence │
            └───┴──┘╰───────╯   ┃               └───┘   ╰───────╯
```

In the first execution, `t_1`’s store synchronizes-with and therefore
happens-before `t_2`’s fence due to the prior load, but note that it does _not_
happen-before `t_2`’s load.

Acquire fences work on any number of atomics, and on release sequences too. A
more complex example is as follows:

```rust
# use std::sync::atomic::{self, AtomicU32};
static X: AtomicU32 = AtomicU32::new(0);
static Y: AtomicU32 = AtomicU32::new(0);

// t_1
X.store(1, atomic::Ordering::Release);
X.fetch_add(1, atomic::Ordering::Relaxed);

// t_2
Y.store(1, atomic::Ordering::Release);

// t_3
let x = X.load(atomic::Ordering::Relaxed);
let y = Y.load(atomic::Ordering::Relaxed);
atomic::fence(atomic::Ordering::Acquire);
```

This can result in an execution like so:

```text
   t_1        X        t_3        Y        t_2
╭───────╮   ┌───┐   ╭───────╮   ┌───┐   ╭───────╮
│ store ├─┐ │ 0 │ ┌─┤ load  │   │ 0 │ ┌─┤ store │
╰───╥───╯ │ └───┘ │ ╰───╥───╯   └───┘ │ ╰───────╯
╭───⇓───╮ └─↘───┐ │ ╭───⇓───╮   ┌───↙─┘
│  rmw  ├─┐ │ 1 │ │ │ load  ├───┤ 1 │
╰───────╯ │ └─┬─┘ │ ╰───╥───╯ ┌─┴───┘
          └─┬─↓─┐ │ ╭───⇓───╮ │
            │ 2 ├─┘┌→ fence ←─┘
            └───┴──┘╰───────╯
```

There are two common scenarios in which acquire fences are used:
1. When an `Acquire` ordering is only necessary when a specific value is loaded.
	For example, you may only wish to acquire when an `initialized` boolean is
	`true`, since otherwise you won’t be reading the shared state at all. In
	this case, you can load with a `Relaxed` ordering and then issue an
	`Acquire` fence afterward only if that condition is met, which can aid in
	performance sometimes (since the acquire operation is avoided when
	`initialized == false`).
2. When several `Acquire` operations on different locations need to be performed
	in a row, but individually each operation doesn’t need `Acquire` ordering;
	it is often faster to perform all the loads as `Relaxed` first and use a
	single `Acquire` fence at the end then it is to make each one separately use
	`Acquire`.

## Release fences

Release fences are the natural complement to acquire fences, and they similarly
can be triggered in three different ways:
1. `atomic::fence(atomic::Ordering::Release)`
1. `atomic::fence(atomic::Ordering::AcqRel)`
1. `atomic::fence(atomic::Ordering::SeqCst)`

Release fences convert every subsequent atomic access in the same thread into a
release operation that has its arrow starting from the fence — in other words,
every `Acquire` operation that sees a value that was written by the fence’s
thread after the release fence will synchronize-with the release fence. For
example, the following code:

```rust
# use std::sync::atomic::{self, AtomicU32};
static X: AtomicU32 = AtomicU32::new(0);

// t_1
atomic::fence(atomic::Ordering::Release);
X.store(1, atomic::Ordering::Relaxed);

// t_2
X.load(atomic::Ordering::Acquire);
```

Can result in this execution:

```text
   t_1        X        t_2
╭───────╮   ┌───┐   ╭───────╮
│ fence ├─┐ │ 0 │ ┌─→ load  │
╰───╥───╯ │ └───┘ │ ╰───────╯
╭───⇓───╮ └─↘───┐ │
│ store ├───┤ 1 ├─┘
╰───────╯   └───┘
```

As well as it being possible for a release fence to synchronize-with an acquire
load (fence–atomic synchronization) and a release store to synchronize-with an
acquire fence (atomic–fence synchronization), it is also possible for release
fences to synchronize with acquire fences (fence–fence synchronization). In this
code snippet, only fences and `Relaxed` operations are used to establish a
happens-before relation (in some executions):

```rust
# use std::sync::atomic::{self, AtomicU32};
static X: AtomicU32 = AtomicU32::new(0);

// t_1
atomic::fence(atomic::Ordering::Release);
X.store(1, atomic::Ordering::Relaxed);

// t_2
X.load(atomic::Ordering::Relaxed);
atomic::fence(atomic::Ordering::Acquire);
```

The execution with the relation looks like this:

```text
   t_1        X        t_2
╭───────╮   ┌───┐   ╭───────╮
│ fence ├─┐ │ 0 │ ┌─┤ load  │
╰───╥───╯ │ └───┘ │ ╰───╥───╯
╭───⇓───╮ └─↘───┐ │ ╭───⇓───╮
│ store ├───┤ 1 ├─┘┌→ fence │
╰───────╯   └───┴──┘╰───────╯
```

Like with acquire fences, release fences can be used to optimize over a series
of atomic stores that don’t individually need to be `Release`, since in some
conditions and on some architectures it’s faster to put a single release fence
at the start and use `Relaxed` from that point on than it is to use `Release`
every time.

## `AcqRel` fences

`AcqRel` fences are just the combined behaviour of an `Acquire` fence and a
`Release` fence in one operation. There isn’t much special to note about them,
other than that they behave more like an acquire fence followed by a release
fence than the other way around, which is useful to know in situations like the
following:

```text
   t_1        X        t_2        Y        t_3
╭───────╮   ┌───┐   ╭───────╮   ┌───┐   ╭───────╮
│   A   │   │ 0 │ ┌─┤ load  │   │ 0 │ ┌─→ load  │
╰───╥───╯   └───┘ │ ╰───╥───╯   └───┘ │ ╰───╥───╯
╭───⇓───╮ ┌─↘───┐ │ ╭───⇓───╮┌──↘───┐ │ ╭───⇓───╮
│ store ├─┘ │ 1 ├─┘┌→ fence ├┘┌─┤ 1 ├─┘ │   B   │
╰───────╯   └───┴──┘╰───╥───╯ │ └───┘   ╰───────╯
                    ╭───⇓───╮ │
                    │ store ├─┘
                    ╰───────╯
```

Here, A happens-before B, which is singularly due to the `AcqRel` fence’s
ability to “carry over” happens-before relations within itself.

## `SeqCst` fences

`SeqCst` fences are the strongest kind of fence. They first of all inherit the
behaviour from an `AcqRel` fence, meaning they have both acquire and release
semantics at the same time, but being `SeqCst` operations they also participate
in _S_. Just as with all other `SeqCst` operations, their placement in _S_ is
primarily determined by strongly happens-before relations (including the
[mixed-`SeqCst` caveat] that comes with it), which then gives additional
guarantees to your code.

Namely, the power of `SeqCst` fences can be summarized in three points:

* Everything that happens-before a `SeqCst` fence is not coherence-ordered-after
	any `SeqCst` operation that the fence precedes in _S_.
* Everything that happens-after a `SeqCst` fence is not coherence-ordered-before
	any `SeqCst` operation that the fence succeeds in _S_.
* Everything that happens-before a `SeqCst` fence X is not
	coherence-ordered-after anything that happens-after another `SeqCst` fence
	Y, if X preceeds Y in _S_.

> In C++11, the above three statements were similar, except they only talked
> about what was sequenced-before and sequenced-after the `SeqCst` fences; C++20
> strengthened this to also include happens-before, because in practice this
> theoretical optimization was not being exploited by anybody. However do note
> that as of the time of writing, [Miri only implements the old, weaker
> semantics][miri scfix] and so you may see false positives when testing with
> it.

The “motivating use-case” for `SeqCst` demonstrated in the `SeqCst` chapter can
also be rewritten to use exclusively `SeqCst` fences and `Relaxed` operations,
by inserting fences in between the operations in the two threads:

```text
     a        static X    static Y         b
╭─────────╮   ┌───────┐   ┌───────┐   ╭─────────╮
│ store X ├─┐ │ false │   │ false │ ┌─┤ store Y │
╰────╥────╯ │ └───────┘   └───────┘ │ ╰────╥────╯
╭────⇓────╮ └─┬───────┐   ┌───────┬─┘ ╭────⇓────╮
│ *fence* │   │ true  │   │ true  │   │ *fence* │
╰────╥────╯   └───────┘   └───────┘   ╰────╥────╯
╭────⇓────╮                           ╭────⇓────╮
│ load Y  ├─?                       ?─┤ load X  │
╰─────────╯                           ╰─────────╯
```

There are two executions to consider here, depending on which way round the
fences appear in _S_. Should `a`’s fence appear first, the fence–fence `SeqCst`
guarantee tells us that `b`’s load of `X` is not coherence-ordered-after `a`’s
store of `X`, which forbids `b`’s load of `X` from seeing the value `false`. The
same logic can be applied should the fences appear the other way around, proving
that at least one thread must load `true` in the end.

[`core::sync::atomic::fence`]: https://doc.rust-lang.org/stable/core/sync/atomic/fn.fence.html
[mixed-`SeqCst` caveat]: seqcst.md#the-mixed-seqcst-special-case
[miri scfix]: https://github.com/rust-lang/miri/issues/2301
