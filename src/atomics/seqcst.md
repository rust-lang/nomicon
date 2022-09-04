# SeqCst

`SeqCst` is probably the most interesting ordering, because it is simultaneously
the simplest and most complex atomic memory ordering in existence. It’s
simple, because if you do only use `SeqCst` everywhere then you can kind of
maybe pretend like the Abstract Machine has a concept of time; phrases like
“latest value” make sense, the program can be thought of as a set of steps that
interleave, there is a universal “now” and “before” and wouldn’t that be nice?
But it’s also the most complex, because as soon as look under the hood you
realize just how incredibly convoluted and hard to follow the actual rules
behind it are, and it gets really ugly really fast as soon as you try to mix it
with any other ordering.

To understand `SeqCst`, we first have to understand the problem it exists to
solve. The first complexity is that this problem can only be observed in the
presence of at least four different threads _and_ two separate atomic variables;
anything less and it’s not possible to notice a difference. The common example
used to show where weaker orderings produce counterintuitive results is this:

```rust
# use std::sync::atomic::{self, AtomicBool};
use std::thread;

// Set this to Relaxed, Acquire, Release, AcqRel, doesn’t matter — the result is
// the same (modulo panics caused by attempting acquire stores or release
// loads).
const ORDERING: atomic::Ordering = atomic::Ordering::Relaxed;

static X: AtomicBool = AtomicBool::new(false);
static Y: AtomicBool = AtomicBool::new(false);

let a = thread::spawn(|| { X.store(true, ORDERING) });
let b = thread::spawn(|| { Y.store(true, ORDERING) });
let c = thread::spawn(|| { while !X.load(ORDERING) {} Y.load(ORDERING) });
let d = thread::spawn(|| { while !Y.load(ORDERING) {} X.load(ORDERING) });

let a = a.join().unwrap();
let b = b.join().unwrap();
let c = c.join().unwrap();
let d = d.join().unwrap();

# return;
// This assert is allowed to fail.
assert!(c || d);
```

The basic setup of this code, for all of its possible executions, looks like
this:

```text
     a        static X         c               d        static Y         b
╭─────────╮   ┌───────┐   ╭─────────╮     ╭─────────╮   ┌───────┐   ╭─────────╮
│ store X ├─┐ │ false │ ┌─┤ load X  │     │ load Y  ├─┐ │ false │ ┌─┤ store Y │
╰─────────╯ │ └───────┘ │ ╰────╥────╯     ╰────╥────╯ │ └───────┘ │ ╰─────────╯
            └─┬───────┐ │ ╭────⇓────╮     ╭────⇓────╮ │ ┌───────┬─┘
              │ true  ├─┘ │ load Y  ├─? ?─┤ load X  │ └─┤ true  │
              └───────┘   ╰─────────╯     ╰─────────╯   └───────┘
```

In other words, `a` and `b` are guaranteed to, at some point, store `true` into
`X` and `Y` respectively, and `c` and `d` are guaranteed to, at some point, load
those values of `true` from `X` and `Y` (there could also be an arbitrary number
of loads of `false` by `c` and `d`, but they’ve been omitted since they don’t
actually affect the execution at all). The question now is when `c` and `d` load
from `Y` and `X` respectively, is it possible for them _both_ to load `false`?

And looking at this diagram, there’s absolutely no reason why not. There isn’t
even a single arrow connecting the left and right hand sides so far, so the load
has no coherence-based restrictions on which value it is allowed to pick — and
this goes for both sides equally, so we could end up with an execution like
this:

```text
     a        static X         c             d        static Y         b
╭─────────╮   ┌───────┐   ╭─────────╮   ╭─────────╮   ┌───────┐   ╭─────────╮
│ store X ├─┐ │ false ├┐ ┌┤ load X  │   │ load Y  ├┐ ┌┤ false │ ┌─┤ store Y │
╰─────────╯ │ └───────┘│ │╰────╥────╯   ╰────╥────╯│ │└───────┘ │ ╰─────────╯
            └─┬───────┐└─│─────║──────┐┌─────║─────│─┘┌───────┬─┘
              │ true  ├──┘╭────⇓────╮┌─┘╭────⇓────╮└──┤ true  │
              └───────┘   │ load Y  ├┘└─┤ load X  │   └───────┘
                          ╰─────────╯   ╰─────────╯
```

Which results in a failed assert. This execution is brought about because the
model of separate modification orders means that there is no relative ordering
between `X` and `Y` being changed, and so each thread is allowed to “see” either
order. However, some algorithms will require a globally agreed-upon ordering,
and this is where `SeqCst` can come in useful.

This ordering, first and foremost, inherits the guarantees from all the other
orderings — it is an acquire operation for loads, a release operation for stores
and an acquire-release operation for RMWs. In addition to this, it gives some
guarantees unique to `SeqCst` about what values it is allowed to load. Note that
these guarantees are not about preventing data races: unless you have some
unrelated code that triggers a data race given an unexpected condition, using
`SeqCst` can only prevent you from race conditions because its guarantees only
apply to other `SeqCst` operations rather than all data accesses.

## S

`SeqCst` is fundamentally about _S_, which is the global ordering of all
`SeqCst` operations in an execution of the program. It is consistent between
every atomic and every thread, and all stores, fences and RMWs that use a
sequentially consistent ordering have a place in it (but no other operations
do). It is in contrast to modification orders, which are similarly total but
only scoped to a single atomic rather than the whole program.

Other than an edge case involving `SeqCst` mixed with weaker orderings (detailed
later on), _S_ is primarily controlled by the happens-before relations in a
program: this means that if an action _A_ happens-before an action _B_, it is
also guaranteed to appear before _B_ in _S_. Other than that restriction, _S_ is
unspecified and will be chosen arbitrarily during execution.

Once a particular _S_ has been established, every atomic’s modification order is
then guaranteed to be consistent with it, so a `SeqCst` load will never see a
value that has been overwritten by a write that occurred before it in _S_, or a
value that has been written by a write that occured after it in _S_ (note that a
`Relaxed`/`Acquire` load however might, since there is no “before” or “after” as
it is not in _S_ in the first place).

More formally, this guarantee can be described with _coherence orderings_, a
relation which expresses which of two operations appears before the other in an
atomic’s modification order. It is said that an operation _A_ is
_coherence-ordered-before_ another operation _B_ if any of the following
conditions are met:
1. _A_ is a store or RMW, _B_ is a store or RMW, and _A_ appears before _B_ in
	the modification order.
1. _A_ is a store or RMW, _B_ is a load, and _B_ reads the value stored by _A_.
1. _A_ is a load, _B_ is a store or RMW, and _A_ takes its value from a place in
	the modification order that appears before _B_.
1. _A_ is coherence-ordered-before a different operation _X_, and _X_ is
	coherence-ordered-before _B_ (the basic transitivity property).

The following diagram gives examples for the main three rules (in each case _A_
is coherence-ordered-before _B_):

```text
        Rule 1        ┃         Rule 2        ┃         Rule 3
                      ┃                       ┃
╭───╮ ┌─┬───┐   ╭───╮ ┃ ╭───╮ ┌─┬───┐   ╭───╮ ┃ ╭───╮   ┌───┐   ╭───╮
│ A ├─┘ │   │ ┌─┤ B │ ┃ │ A ├─┘ │   ├───┤ B │ ┃ │ A ├───┤   │ ┌─┤ B │
╰───╯   └───┘ │ ╰───╯ ┃ ╰───╯   └───┘   ╰───╯ ┃ ╰───╯   └───┘ │ ╰───╯
        ┌───┬─┘       ┃                       ┃         ┌───┬─┘
        │   │         ┃                       ┃         │   │
        └───┘         ┃                       ┃         └───┘
```

The only important thing to note is that for two loads of the same value in the
modification order, neither is coherence-ordered-before the other, as in the
following example where _A_ has no coherence ordering relation to _B_:

```text
╭───╮   ┌───┐   ╭───╮
│ A ├───┤   ├───┤ B │
╰───╯   └───┘   ╰───╯
```

Because of this, “_A_ is coherence-ordered-before _B_” is subtly different from
“_A_ is not coherence-ordered-after _B_”; only the latter phrase includes the
above situation, and is synonymous with “either _A_ is coherence-ordered-before
_B_ or _A_ and _B_ are both loads, and see the same value in the atomic’s
modification order”. “Not coherence-ordered-after” is generally a more useful
relation than “coherence-ordered-before”, and so it’s important to understand
what it means.

With this terminology applied, we can use a more precise definition of
`SeqCst`’s guarantee: for two `SeqCst` operations on the same atomic _A_ and
_B_, where _A_ precedes _B_ in _S_, _A_ is not coherence-ordered-after _B_.
Effectively, this one rule ensures that _S_’s order “propagates”
throughout all the atomics of the program — you can imagine each operation in
_S_ as storing a snapshot of the world, so that every subsequent operation is
consistent with it.

## Applying `SeqCst`

So, looking back at our program, let’s consider how we could use `SeqCst` to
make that execution invalid. As a refresher, here’s the framework for every
possible execution of the program:

```text
     a        static X         c               d        static Y         b
╭─────────╮   ┌───────┐   ╭─────────╮     ╭─────────╮   ┌───────┐   ╭─────────╮
│ store X ├─┐ │ false │ ┌─┤ load X  │     │ load Y  ├─┐ │ false │ ┌─┤ store Y │
╰─────────╯ │ └───────┘ │ ╰────╥────╯     ╰────╥────╯ │ └───────┘ │ ╰─────────╯
            └─┬───────┐ │ ╭────⇓────╮     ╭────⇓────╮ │ ┌───────┬─┘
              │ true  ├─┘ │ load Y  ├─? ?─┤ load X  │ └─┤ true  │
              └───────┘   ╰─────────╯     ╰─────────╯   └───────┘
```

First of all, both the final loads (`c` and `d`’s second operations) need to
become `SeqCst`, because they need to be aware of the total ordering that
determines whether `X` or `Y` becomes `true` first. And secondly, we need to
establish that ordering in the first place, and that needs to be done by making
sure that there is always one operation in _S_ that both sees one of the atomics
as `true` and precedes both final loads in _S_, so that the coherence ordering
guarantee will apply (the final loads themselves don’t work for this since
although they “know” that their corresponding atomic is `true` they don’t
interact with it directly so _S_ doesn’t care).

There are two operations in the program that could fulfill the first condition,
should they be made `SeqCst`: the stores of `true` and the first loads. However,
the second condition ends up ruling out using the stores, since in order to make
sure that they precede the final loads in _S_ it would be necessary to have the
first loads be `SeqCst` anyway (due to the mixed-`SeqCst` special case detailed
later), so in the end we can just leave them as `Relaxed`.

This leaves us with the correct version of the above program, which is
guaranteed to never panic:

```rust
# use std::sync::atomic::{AtomicBool, Ordering::{Relaxed, SeqCst}};
use std::thread;

static X: AtomicBool = AtomicBool::new(false);
static Y: AtomicBool = AtomicBool::new(false);

let a = thread::spawn(|| { X.store(true, Relaxed) });
let b = thread::spawn(|| { Y.store(true, Relaxed) });
let c = thread::spawn(|| { while !X.load(SeqCst) {} Y.load(SeqCst) });
let d = thread::spawn(|| { while !Y.load(SeqCst) {} X.load(SeqCst) });

let a = a.join().unwrap();
let b = b.join().unwrap();
let c = c.join().unwrap();
let d = d.join().unwrap();

// This assert is **not** allowed to fail.
assert!(c || d);
```

As there are four `SeqCst` operations with a partial order between two pairs in
them (caused by the sequenced-before relation), there are six possible
executions of this program:

- All of `c`’s loads precede `d`’s loads:
	1. `c` loads `X` (gives `true`)
	1. `c` loads `Y` (gives either `false` or `true`)
	1. `d` loads `Y` (gives `true`)
	1. `d` loads `X` (required to be `true`)
- Both initial loads precede both final loads:
	1. `c` loads `X` (gives `true`)
	1. `d` loads `Y` (gives `true`)
	1. `c` loads `Y` (required to be `true`)
	1. `d` loads `X` (required to be `true`)
- As above, but the final loads occur in a different order:
	1. `c` loads `X` (gives `true`)
	1. `d` loads `Y` (gives `true`)
	1. `d` loads `X` (required to be `true`)
	1. `c` loads `Y` (required to be `true`)
- As before, but the initial loads occur in a different order:
	1. `d` loads `Y` (gives `true`)
	1. `c` loads `X` (gives `true`)
	1. `c` loads `Y` (required to be `true`)
	1. `d` loads `X` (required to be `true`)
- As above, but the final loads occur in a different order:
	1. `d` loads `Y` (gives `true`)
	1. `c` loads `X` (gives `true`)
	1. `d` loads `X` (required to be `true`)
	1. `c` loads `Y` (required to be `true`)
- All of `d`’s loads precede `c`’s loads:
	1. `d` loads `Y` (gives `true`)
	1. `d` loads `X` (gives either `false` or `true`)
	1. `c` loads `X` (gives `true`)
	1. `c` loads `Y` (required to be `true`)

All the places where the load was required to give `true` were caused by a
preceding load in _S_ of the same atomic which saw `true` — otherwise, the load
would be coherence-ordered-before a load which precedes it in _S_, and that is
impossible.

## The mixed-`SeqCst` special case

As I’ve been alluding to for a while, I wasn’t being totally truthful when I
said that _S_ is consistent with happens-before relations — in reality, it is
only consistent with _strongly happens-before_ relations, which presents a
subtly-defined subset of happens-before relations. In particular, it excludes
two situations:

1. The `SeqCst` operation A synchronizes-with an `Acquire` or `AcqRel` operation
   B which is sequenced-before another `SeqCst` operation C. Here, despite the
   fact that A happens-before C, A does not _strongly_ happen-before C and so is
   there not guaranteed to precede C in _S_.
2. The `SeqCst` operation A is sequenced-before the `Release` or `AcqRel`
   operation B, which synchronizes-with another `SeqCst` operation C. Similarly,
   despite the fact that A happens-before C, A might not precede C in _S_.

The first situation is illustrated below, with `SeqCst` accesses repesented with
asterisks:

```text
  t_1       x       t_2
╭─────╮ ┌─↘───┐   ╭─────╮
│ *A* ├─┘ │ 1 ├───→  B  │
╰─────╯   └───┘   ╰──╥──╯
                  ╭──⇓──╮
                  │ *C* │
                  ╰─────╯
```

A happens-before, but does not strongly happen-before, C — and anything
sequenced-after C will have the same treatment (unless more synchronization is
used). This means that C is actually allowed to _precede_ A in _S_, despite
conceptually occuring after it. However, anything sequenced-before A, because
there is at least one sequence on either side of the synchronization, will
strongly happen-before C.

But this is all highly theoretical at the moment, so let’s make an example to
show how that rule can actually affect the execution of code. So, if C were to
precede A in _S_ (and they are not both loads) then that means C is always
coherence-ordered-before A. Let’s say then that C loads from `x` (the atomic
that A has to access), it may load the value that came before A if it were to
precede A in _S_:

```text
  t_1       x       t_2
╭─────╮   ┌───┐   ╭─────╮
│ *A* ├─┐ │ 0 ├─┐┌→  B  │
╰─────╯ │ └───┘ ││╰──╥──╯
        └─↘───┐┌─┘╭──⇓──╮
          │ 1 ├┘└─→ *C* │
          └───┘   ╰─────╯
```

Ah wait no, that doesn’t work because regular coherence still mandates that `1`
is the only value that can be loaded. In fact, once `1` is loaded _S_’s required
consistency with coherence orderings means that A _is_ required to precede C in
_S_ after all.

So somehow, to observe this difference we need to have a _different_ `SeqCst`
operation, let’s call it E, be the one that loads from `x`, where C is
guaranteed to precede it in _S_ (so we can observe the “weird” state in between
C and A) but C also doesn’t happen-before it (to avoid coherence getting in the
way) — and to do that, all we have to do is have C appear before a `SeqCst`
operation D in the modification order of another atomic, but have D be a store
so as to avoid C synchronizing with it, and then our desired load E can simply
be sequenced-after D (this will carry over the “precedes in _S_” guarantee, but
does not restore the happens-after relation to C since that was already dropped
by having D be a store).

In diagram form, that looks like this:

```text
  t_1       x       t_2     helper      t_3
╭─────╮   ┌───┐   ╭─────╮   ┌─────┐   ╭─────╮
│ *A* ├─┐ │ 0 ├┐┌─→  B  │ ┌─┤  0  │ ┌─┤ *D* │
╰─────╯ │ └───┘││ ╰──╥──╯ │ └─────┘ │ ╰──╥──╯
        │      └│────║────│─────────│┐   ║
        └─↘───┐ │ ╭──⇓──╮ │ ┌─────↙─┘│╭──⇓──╮
          │ 1 ├─┘ │ *C* ←─┘ │  1  │  └→ *E* │
          └───┘   ╰─────╯   └─────┘   ╰─────╯

S = C → D → E → A
```

C is guaranteed to precede D in _S_, and D is guaranteed to precede E, but
because this exception means that A is _not_ guaranteed to precede C, it is
totally possible for it to come at the end, resulting in the surprising but
totally valid outcome of E loading `0` from `x`. In code, this can be expressed
as the following code _not_ being guaranteed to panic:

```rust
# use std::sync::atomic::{AtomicU8, Ordering::{Acquire, SeqCst}};
# return;
static X: AtomicU8 = AtomicU8::new(0);
static HELPER: AtomicU8 = AtomicU8::new(0);

// thread_1
X.store(1, SeqCst); // A

// thread_2
assert_eq!(X.load(Acquire), 1); // B
assert_eq!(HELPER.load(SeqCst), 0); // C

// thread_3
HELPER.store(1, SeqCst); // D
assert_eq!(X.load(SeqCst), 0); // E
```

The second situation listed above has very similar consequences. Its abstract
form is the following execution in which A is not guaranteed to precede C in
_S_, despite A happening-before C:

```text
  t_1       x       t_2
╭─────╮ ┌─↘───┐   ╭─────╮
│ *A* │ │ │ 0 ├───→ *C* │
╰──╥──╯ │ └───┘   ╰─────╯
╭──⇓──╮ │
│  B  ├─┘
╰─────╯
```

Similarly to before, we can’t just have A access `x` to show why A not
necessarily preceding C in _S_ matters; instead, we have to introduce a second
atomic and third thread to break the happens-before chain first. And finally, a
single relaxed load F at the end is added just to prove that the weird execution
actually happened (leaving `x` as 2 instead of 1).

```text
  t_3     helper      t_1       x       t_2
╭─────╮   ┌─────┐   ╭─────╮   ┌───┐   ╭─────╮
│ *D* ├┐┌─┤  0  │ ┌─┤ *A* │   │ 0 │ ┌─→ *C* │
╰──╥──╯││ └─────┘ │ ╰──╥──╯   └───┘ │ ╰──╥──╯
   ║   └│─────────│────║─────┐      │    ║
╭──⇓──╮ │ ┌─────↙─┘ ╭──⇓──╮ ┌─↘───┐ │ ╭──⇓──╮
│ *E* ←─┘ │  1  │   │  B  ├─┘││ 1 ├─┘┌┤  F  │
╰─────╯   └─────┘   ╰─────╯  │└───┘  │╰─────╯
                             └↘───┐  │
                              │ 2 ├──┘
                              └───┘
S = C → D → E → A
```

This execution mandates both C preceding A in _S_ and A happening-before C,
something that is only possible through these two mixed-`SeqCst` special
exceptions. It can be expressed in code as well:

```rust
# use std::sync::atomic::{AtomicU8, Ordering::{Release, Relaxed, SeqCst}};
# return;
static X: AtomicU8 = AtomicU8::new(0);
static HELPER: AtomicU8 = AtomicU8::new(0);

// thread_3
X.store(2, SeqCst); // D
assert_eq!(HELPER.load(SeqCst), 0); // E

// thread_1
HELPER.store(1, SeqCst); // A
X.store(1, Release); // B

// thread_2
assert_eq!(X.load(SeqCst), 1); // C
assert_eq!(X.load(Relaxed), 2); // F
```

If this seems ridiculously specific and obscure, that’s because it is.
Originally, back in C++11, this special case didn’t exist — but then six years
later it was discovered that in practice atomics on Power, Nvidia GPUs and
sometimes ARMv7 _would_ have this special case, and fixing the implementations
would make atomics significantly slower. So instead, in C++20 they simply
encoded it into the specification.

Generally however, this rule is so complex it’s best to just avoid it entirely
by never mixing `SeqCst` and non-`SeqCst` on a single atomic in the first place.
