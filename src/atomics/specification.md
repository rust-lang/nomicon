# Specification

Below is a modified C++20 specification draft (as it was on 2022-07-16), edited
to remove C++-only features like consume orderings and `sig_atomic_t`.

Note that although this has been checked, atomics are very difficult to get
right and so there may be subtle mistakes. If you want to more formally check
your software, read the [\[intro.races\]], [\[atomics.order\]] and
[\[atomics.fences\]] sections of the real C++ specification.

[\[intro.races\]]: https://eel.is/c++draft/intro.races
[\[atomics.order\]]: https://eel.is/c++draft/atomics.order
[\[atomics.fences\]]: https://eel.is/c++draft/atomics.fences

## Data races

The value of an object visible to a thread _T_ at a particular point is the
initial value of the object, a value assigned to the object by _T_, or a value
assigned to the object by another thread, according to the rules below.

> _Note 1_: In some cases, there might instead be undefined behavior. Much of
> this subclause is motivated by the desire to support atomic operations with
> explicit and detailed visibility constraints. However, it also implicitly
> supports a simpler view for more restricted programs.

Two expression evaluations _conflict_ if one of them modifies a memory location
and the other one reads or modifies the same memory location.

The library defines a number of atomic operations and operations on mutexes that
are specially identified as synchronization operations. These operations play a
special role in making assignments in one thread visible to another. A
synchronization operation on one or more memory locations is either an acquire
operation, a release operation, or both an acquire and release operation. A
synchronization operation without an associated memory location is a fence and
can be either an acquire fence, a release fence, or both an acquire and release
fence. In addition, there are relaxed atomic operations, which are not
synchronization operations, and atomic read-modify-write operations, which have
special characteristics.

> _Note 2_: For example, a call that acquires a mutex will perform an acquire
> operation on the locations comprising the mutex. Correspondingly, a call that
> releases the same mutex will perform a release operation on those same
> locations. Informally, performing a release operation on _A_ forces prior side
> effects on other memory locations to become visible to other threads that
> later perform an acquire operation on _A_. “Relaxed” atomic operations are not
> synchronization operations even though, like synchronization operations, they
> cannot contribute to data races.

All modifications to a particular atomic object _M_ occur in some particular
total order, called the _modification order_ of _M_.

> _Note 3_: There is a separate order for each atomic object. There is no
> requirement that these can be combined into a single total order for all
> objects. In general this will be impossible since different threads can
> observe modifications to different objects in inconsistent orders.

A _release sequence_ headed by a release operation _A_ on an atomic object _M_
is a maximal contiguous sub-sequence of side effects in the modification order
of _M_, where the first operation is _A_, and every subsequent operation is an
atomic read-modify-write operation.

Certain library calls _synchronize with_ other library calls performed by
another thread. For example, an atomic store-release synchronizes with a
load-acquire that takes its value from the store.

> _Note 4_: Except in the specified cases, reading a later value does not
> necessarily ensure visibility as described below. Such a requirement would
> sometimes interfere with efficient implementation.

> _Note 5_: The specifications of the synchronization operations define when one
> reads the value written by another. For atomic objects, the definition is
> clear. All operations on a given mutex occur in a single total order. Each
> mutex acquisition “reads the value written” by the last mutex release.

An evaluation _A_ _happens before_ an evaluation _B_ (or, equivalently, _B_
_happens after_ _A_) if either:
- _A_ is sequenced before _B_, or
- _A_ synchronizes with _B_, or
- for some evaluation _X_, _A_ happens before _X_ and _X_ happens before _B_.

An evaluation _A_ _strongly happens before_ an evaluation _D_ if, either
- _A_ is sequenced before _D_, or
- _A_ synchronizes with _D_, and both _A_ and _D_ and sequentially consistent
	atomic operations, or
- there are evaluations _B_ and _C_ such that _A_ is sequenced before _B_, _B_
	happens before _C_, and _C_ is sequenced before _D_, or
- there is an evaluation _B_ such that _A_ strongly happens before _B_, and _B_
	strongly happens before _D_.

> _Note 11_: Informally, if _A_ strongly happens before _B_, then _A_ appears to
> be evaluated before _B_ in all contexts.

A _visible side effect_ _A_ on a scalar object _M_ with respect to a value
computation _B_ of _M_ satisfies the conditions:
- _A_ happens before _B_ and
- there is no other side effect _X_ to _M_ such that _A_ happens before _X_ and
	_X_ happens before _B_.

The value of a non-atomic scalar object _M_, as determined by evaluation _B_,
shall be the value stored by the visible side effect _A_.

> _Note 12_: If there is ambiguity about which side effect to a non-atomic
> object is visible, then the behavior is either unspecified or undefined.

> _Note 13_: This states that operations on ordinary objects are not visibly
> reordered. This is not actually detectable without data races, but it is
> necessary to ensure that data races, as defined below, and with suitable
> restrictions on the use of atomics, correspond to data races in a simple
> interleaved (sequentially consistent) execution.

The value of an atomic object _M_, as determined by evaluation _B_, shall be the
value stored by some side effect _A_ that modifies _M_, where _B_ does not
happen before _A_.

> _Note 14_: The set of such side effects is also restricted by the rest of the
> rules described here, and in particular, by the coherence requirements below.

If an operation _A_ that modifies an atomic object _M_ happens before an
operation _B_ that modifies _M_, then _A_ shall be earlier than _B_ in the
modification order of _M_.

> _Note 15_: This requirement is known as write-write coherence.

If a value computation _A_ of an atomic object _M_ happens before a value
computation _B_ of _M_, and _A_ takes its value from a side effect _X_ on _M_,
then the value computed by _B_ shall either be the value stored by _X_ or the
value stored by a side effect _Y_ on _M_, where _Y_ follows _X_ in the
modification order of _M_.

> _Note 16_: This requirement is known as read-read coherence.

If a value computation _A_ of an atomic object _M_ happens before an operation
_B_ that modifies _M_, then _A_ shall take its value from a side effect _X_ on
_M_, where _X_ precedes _B_ in the modification order of _M_.

> _Note 17_: This requirement is known as read-write coherence.

If a side effect _X_ on an atomic object _M_ happens before a value computation
_B_ of _M_, then the evaluation _B_ shall take its value from _X_ or from a side
effect _Y_ that follows _X_ in the modification order of _M_.

> _Note 18_: This requirement is known as write-read coherence.

> _Note 19_: The four preceding coherence requirements effectively disallow
> compiler reordering of atomic operations to a single object, even if both
> operations are relaxed loads. This effectively makes the cache coherence
> guarantee provided by most hardware available to C++ atomic operations.

> _Note 20_: The value observed by a load of an atomic depends on the “happens
> before” relation, which depends on the values observed by loads of atomics.
> The intended reading is that there must exist an association of atomic loads
> with modifications they observe that, together with suitably chosen
> modification orders and the “happens before” relation derived as described
> above, satisfy the resulting constraints as imposed here.

Two actions are _potentially concurrent_ if
- they are performed by different threads, or
- they are unsequenced, at least one is performed by a signal handler, and they
	are not both performed by the same signal handler invocation.

The execution of a program contains a _data race_ if it contains two potentially
concurrent conflicting actions, at least one of which is not atomic, and neither
happens before the other. Any such data race results in undefined behavior.

> _Note 21_: It can be shown that programs that correctly use mutexes and
> `SeqCst` operations to prevent all data races and use no other synchronization
> operations behave as if the operations executed by their constituent threads
> were simply interleaved, with each value computation of an object being taken
> from the last side effect on that object in that interleaving. This is normally
> referred to as “sequential consistency”. However, this applies only to
> data-race-free programs, and data-race-free programs cannot observe most
> program transformations that do not change single-threaded program semantics.
> In fact, most single-threaded program transformations continue to be allowed,
> since any program that behaves differently as a result has undefined behavior.

> _Note 22_: Compiler transformations that introduce assignments to a
> potentially shared memory location that would not be modified by the abstract
> machine are generally precluded by this document, since such an assignment
> might overwrite another assignment by a different thread in cases in which an
> abstract machine execution would not have encountered a data race. This
> includes implementations of data member assignment that overwrite adjacent
> members in separate memory locations. Reordering of atomic loads in cases in
> which the atomics in question might alias is also generally precluded, since
> this could violate the coherence rules.

> _Note 23_: Transformations that introduce a speculative read of a potentially
> shared memory location might not preserve the semantics of the C++ program as
> defined in this document, since they potentially introduce a data race.
> However, they are typically valid in the context of an optimizing compiler
> that targets a specific machine with well-defined semantics for data races.
> They would be invalid for a hypothetical machine that is not tolerant of races
> or provides hardware race detection. 

## Atomic orderings

```rust
// in ::core::sync::atomic
#[non_exhaustive]
pub enum Ordering {
	Relaxed,
	Release,
	Acquire,
	AcqRel,
	SeqCst,
}
```

The enumeration `Ordering` specifies the detailed regular (non-atomic) memory
synchronization order as defined in this document and may provide for operation
ordering. Its enumerated values and their meanings are as follows:
- `Relaxed`: no operation orders memory.
- `Release`, `AcqRel`, and `SeqCst`: a store operation performs a release
	operation on the affected memory location.
- `Acquire`, `AcqRel`, and `SeqCst`: a load operation performs an acquire
	operation on the affected memory location.

> _Note 2_: Atomic operations specifying `Relaxed` are relaxed with respect to
> memory ordering. Implementations must still guarantee that any given atomic
> access to a particular atomic object be indivisible with respect to all other
> atomic accesses to that object.

An atomic operation _A_ that performs a release operation on an atomic object
_M_ synchronizes with an atomic operation _B_ that performs an acquire operation
on _M_ and takes its value from any side effect in the release sequence headed
by _A_.

An atomic operation _A_ on some atomic object _M_ is coherence-ordered before
another atomic operation _B_ on _M_ if
- _A_ is a modification, and _B_ reads the value stored by _A_, or
- _A_ precedes _B_ in the modification order of _M_, or
- _A_ and _B_ are not the same atomic read-modify-write operation, and there
	exists an atomic modification _X_ of _M_ such that _A_ reads the value
	stored by _X_ and _X_ precedes _B_ in the modification order of _M_, or
- there exists an atomic modification _X_ of _M_ such that _A_ is
	coherence-ordered before _X_ and _X_ is coherence-ordered before _B_.

There is a single total order _S_ on all `SeqCst` operations, including fences,
that satisfies the following constraints. First, if _A_ and _B_ are `SeqCst`
operations and _A_ strongly happens before _B_, then _A_ precedes _B_ in _S_.
Second, for every pair of atomic operations _A_ and _B_ on an object _M_, where
_A_ is coherence-ordered before _B_, the following four conditions are required
to be satisfied by _S_:
- if _A_ and _B_ are both `SeqCst` operations, then _A_ precedes _B_ in _S_; and
- if _A_ is a `SeqCst` operation and _B_ happens before a `SeqCst` fence _Y_,
	then _A_ precedes _Y_ in _S_; and
- if a `SeqCst` fence _X_ happens before _A_ and _B_ is a `SeqCst` operation,
	then _X_ precedes _B_ in _S_; and
- if an `SeqCst` fence _X_ happens before _A_ and _B_ happens before a `SeqCst`
	fence _Y_, then _X_ precedes _Y_ in _S_.

> _Note 3_: This definition ensures that _S_ is consistent with the modification
> order of any atomic object _M_. It also ensures that a `SeqCst` load _A_ of
> _M_ gets its value either from the last modification of _M_ that precedes _A_
> in _S_ or from some non-`SeqCst` modification of _M_ that does not happen
> before any modification of _M_ that precedes _A_ in _S_.

> _Note 4_: We do not require that _S_ be consistent with “happens before”. This
> allows more efficient implementation of `Acquire` and `Release` on some
> machine architectures. It can produce surprising results when these are mixed
> with `SeqCst` accesses. 

> _Note 5_: `SeqCst` ensures sequential consistency only for a program that is
> free of data races and uses exclusively `SeqCst` atomic operations. Any use of
> weaker ordering will invalidate this guarantee unless extreme care is used. In
> many cases, `SeqCst` atomic operations are reorderable with respect to other
> atomic operations performed by the same thread.

Implementations should ensure that no “out-of-thin-air” values are computed that
circularly depend on their own computation.

> _Note 6_: For example, with `x` and `y` initially zero,
> ```rust,ignore
> // Thread 1:
> let r1 = y.load(atomic::Ordering::Relaxed);
> x.store(r1, atomic::Ordering::Relaxed);
> // Thread 2:
> let r2 = x.load(atomic::Ordering::Relaxed);
> y.store(r2, atomic::Ordering::Relaxed);
> ```
> this recommendation discourages producing `r1 == r2 == 42`, since the store of
> 42 to `y` is only possible if the store to `x` stores `42`, which circularly
> depends on the store to `y` storing `42`. Note that without this restriction,
> such an execution is possible.

> _Note 7_: The recommendation similarly disallows `r1 == r2 == 42` in the
> following example, with `x` and `y` again initially zero:
> ```rust,ignore
> // Thread 1:
> let r1 = x.load(atomic::Ordering::Relaxed);
> if r1 == 42 {
>     y.store(42, atomic::Ordering::Relaxed);
> }
> // Thread 2:
> let r2 = y.load(atomic::Ordering::Relaxed);
> if r2 == 42 {
>     x.store(42, atomic::Ordering::Relaxed);
> }
> ```

Atomic read-modify-write operations shall always read the last value (in the
modification order) written before the write associated with the
read-modify-write operation.

Implementations should make atomic stores visible to atomic loads within a
reasonable amount of time.

## Atomic fences

This subclause introduces synchronization primitives called _fences_. Fences can
have acquire semantics, release semantics, or both. A fence with acquire
semantics is called an _acquire fence_. A fence with release semantics is called
a _release fence_.

A release fence _A_ synchronizes with an acquire fence _B_ if there exist atomic
operations _X_ and _Y_, both operating on some atomic object _M_, such that _A_
is sequenced before _X_, _X_ modifies _M_, _Y_ is sequenced before _B_, and _Y_
reads the value written by _X_ or a value written by any side effect in the
hypothetical release sequence _X_ would head if it were a release operation.

A release fence _A_ synchronizes with an atomic operation _B_ that performs an
acquire operation on an atomic object _M_ if there exists an atomic operation
_X_ such that _A_ is sequenced before _X_, _X_ modifies _M_, and _B_ reads the
value written by _X_ or a value written by any side effect in the hypothetical
release sequence _X_ would head if it were a release operation.

An atomic operation _A_ that is a release operation on an atomic object _M_
synchronizes with an acquire fence _B_ if there exists some atomic operation _X_
on _M_ such that _X_ is sequenced before _B_ and reads the value written by _A_
or a value written by any side effect in the release sequence headed by _A_.

```rust,ignore
pub fn fence(order: Ordering);
```

_Effects_: Depending on the value of `order`, this operation:
- has no effects, if `order == Relaxed`;
- is an acquire fence, if `order == Acquire`;
- is a release fence, if `order == Release`;
- is both an acquire and a release fence, if `order == AcqRel`;
- is a sequentially consistent acquire and release fence, if `order == SeqCst`.

```rust,ignore
pub fn compiler_fence(order: Ordering);
```

_Effects_: Equivalent to `fence(order)`, except that the resulting ordering
constraints are established only between a thread and a signal handler executed
in the same thread.

> _Note 1_: `compiler_fence` can be used to specify the order in which actions
> performed by the thread become visible to the signal handler. Compiler
> optimizations and reorderings of loads and stores are inhibited in the same
> way as with `fence` but the hardware fence instructions that `fence` would
> have inserted are not emitted.
