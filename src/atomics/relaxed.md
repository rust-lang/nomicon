# Relaxed

Now we’ve got single-threaded mutation semantics out of the way, we can try
reintroducing a second thread. We’ll have one thread perform a write to the
memory location, and a second thread read from it, like so:

```rust
// Initial state
let mut data = 0;
// Thread 1:
data = 1;
// Thread 2:
println!("{data}");
```

Of course, any Rust programmer will immediately tell you that this code doesn’t
compile, and indeed it definitely does not, and for good reason. But suspend
your disbelief for a moment, and imagine what would happen if it did. Let’s draw
a diagram, leaving out the reading lines for now:

```text
Thread 1     data    Thread 2
╭───────╮   ┌────┐   ╭───────╮
│  = 1  ├╌┐ │  0 │ ?╌┤  data │
╰───────╯ ├╌┼╌╌╌╌┤   ╰───────╯
          └╌┼╌╌╌╌┤
            │  1 │
            └────┘
```

Unfortunately, the rules from before don’t help us in finding out where Thread
2’s line joins up to, since there are no arrows connecting that operation to
anything and therefore we can’t immediately rule any values out. As a result, we
end up facing a situation we haven’t faced before: there is _more than one_
potential value for Thread 2 to read.

And this is where we encounter the big limitation with unsynchronized data
accesses: the price we pay for their speed and optimization capability is that
this situation is considered **Undefined Behavior**. For an unsynchronized read
to be acceptable, there has to be _exactly one_ potential value for it to read,
and when there are multiple like in this situation it is considered a data race.

So what can we do about this? Well, two things need to be changed. First of all,
Thread 1 has to use an atomic store instead of an unsynchronized write, and
secondly Thread 2 has to use an atomic load instead of an unsynchronized read.
You’ll also notice that all the atomic functions accept one (and sometimes two)
parameters of `atomic::Ordering`s — we’ll explore the details of the differences
between them later, but for now we’ll use `Relaxed` because it is by far the
simplest of the lot.

```rust
# use std::sync::atomic::{self, AtomicU32};
// Initial state
let data = AtomicU32::new(0);
// Thread 1:
data.store(1, atomic::Ordering::Relaxed);
// Thread 2:
data.load(atomic::Ordering::Relaxed);
```

The use of the atomic store provides one additional ability in comparison to an
unsynchronized store, and that is that there is no “in-between” state between
the old and new values — instead, it immediately updates, resulting in a diagram
that look a bit more like this:

```text
Thread 1     data
╭───────╮   ┌────┐
│  = 1  ├─┐ │  0 │
╰───────╯ │ └────┘
          └─┬────┐
            │  1 │
            └────┘
```

We have now established a _modification order_ for `data`: a total, ordered list
of distinct, separated values that it takes over its lifetime.

On the loading side, we also obtain one additional ability: when there are
multiple possible values to choose from in the modification order, instead of it
triggering UB, exactly one (but it is unspecified which) value is chosen. This
means that there are now _two_ potential executions of our program, with no way
for us to control which one occurs:

```text
     Possible Execution 1       ┃       Possible Execution 2
                                ┃
Thread 1     data    Thread 2   ┃  Thread 1     data    Thread 2
╭───────╮   ┌────┐   ╭───────╮  ┃  ╭───────╮   ┌────┐   ╭───────╮
│ store ├─┐ │  0 ├───┤  load │  ┃  │ store ├─┐ │  0 │ ┌─┤  load │
╰───────╯ │ └────┘   ╰───────╯  ┃  ╰───────╯ │ └────┘ │ ╰───────╯
          └─┬────┐              ┃            └─┬────┐ │
            │  1 │              ┃              │  1 ├─┘
            └────┘              ┃              └────┘
```

Note that **both sides must be atomic to avoid the data race**: if only the
writing side used atomic operations, the reading side would still have multiple
values to choose from (UB), and if only the reading side used atomic operations
it could end up reading the garbage data “in-between” `0` and `1` (also UB).

> **NOTE:** This description of why both sides are needed to be atomic
> operations, while neat and intuitive, is not strictly correct: in reality the
> answer is simply “because the spec says so”. However, it is isomorphic to the
> real rules, so it can aid in understanding.

## Read-modify-write operations

Loads and stores are pretty neat in avoiding data races, but you can’t get very
far with them. For example, suppose you wanted to implement a global shared
counter that can be used to assign unique IDs to objects. Naïvely, you might try
to write code like this:

```rust
# use std::sync::atomic::{self, AtomicU64};
static COUNTER: AtomicU64 = AtomicU64::new(0);
pub fn get_id() -> u64 {
    let value = COUNTER.load(atomic::Ordering::Relaxed);
    COUNTER.store(value + 1, atomic::Ordering::Relaxed);
    value
}
```

But then calling that function from multiple threads opens you up to an
execution like below that results in two threads obtaining the same ID:

```text
Thread 1   COUNTER   Thread 2
╭───────╮   ┌───┐   ╭───────╮
│ load  ├───┤ 0 ├───┤  load │
╰───╥───╯   └───┘   ╰────╥──╯
╭───⇓───╮ ┌─┬───┐   ╭────⇓──╮
│ store ├─┘ │ 1 │ ┌─┤ store │
╰───────╯   └───┘ │ ╰───────╯
            ┌───┬─┘
            │ 1 │
            └───┘
```

Technically, I believe it is _possible_ to implement this kind of thing with
just loads and stores, if you try hard enough and use several atomics. But
luckily, you don’t have to because there also exists another kind of operation,
the read-modify-write, which is specifically suited to this purpose.

A read-modify-write operation (shortened to RMW) is a special kind of atomic
operation that reads, changes and writes back a value _in one step_. This means
that there are guaranteed to exist no other values in the modification order in
between the read and the write; it happens as a single operation. I would also
like to point out that this is true of **all** atomic orderings, since a common
misconception is that the `Relaxed` ordering somehow negates this guarantee.

There are many different RMW operations to choose from, but the one most
appropriate for this use case is `fetch_add`, which adds a number to the atomic,
as well as returns the old value. So our code can be rewritten as this:

```rust
# use std::sync::atomic::{self, AtomicU64};
static COUNTER: AtomicU64 = AtomicU64::new(0);
pub fn get_id() -> u64 {
    COUNTER.fetch_add(1, atomic::Ordering::Relaxed)
}
```

And then, no matter how many threads there are, that race condition from earlier
can never occur. Executions will have to look more like this:

```text
  Thread 1     COUNTER     Thread 2
╭───────────╮   ┌───┐   ╭───────────╮
│ fetch_add ├─┐ │ 0 │ ┌─┤ fetch_add │
╰───────────╯ │ └───┘ │ ╰───────────╯
              └─┬───┐ │
                │ 1 │ │
                └───┘ │
                ┌───┬─┘
                │ 2 │
                └───┘
```

There is one problem with this code however, and that is that if `get_id()` is
called over 18 446 744 073 709 551 615 times, the counter will overflow and it
will start generating duplicate IDs. Of course, this won’t feasibly happen, but
it can be problematic if you need to _prove_ that it can’t happen (e.g. for
safety purposes) or you’re using a smaller integer type like `u32`.

So we’re going to modify this function so that instead of returning a plain
`u64` it returns an `Option<u64>`, where `None` is used to indicate that an
overflow occurred and no more IDs could be generated. Additionally, it’s not
enough to just return `None` once, because if there are multiple threads
involved they will not see that result if it just occurs on a single thread —
instead, it needs to continue to return `None` _until the end of time_ (or,
well, this execution of the program).

That means we have to do away with `fetch_add`, because `fetch_add` will always
overflow and there’s no `checked_fetch_add` equivalent. We’ll return to our racy
algorithm for a minute, this time thinking more about what went wrong. The steps
look something like this:

1. Load a value of the atomic
1. Perform the checked add, propagating `None`
1. Store in the new value of the atomic

The problem here is that the store does not necessarily occur directly after the
load in the atomic’s modification order, and that leads to the races. What we
need is some way to say, “add this new value to the modification order, but
_only if_ it occurs directly after the value we loaded”. And luckily for us,
there exists a function that does exactly\* this: `compare_exchange`.

`compare_exchange` is a bit like a store, but instead of unconditionally storing
the value, it will first check the previous value in the modification order to
see whether it is what we expect, and if not it will simply tell us that and not
make any changes. It is an RMW operation, so all of this happens fully
atomically — there is no chance for a race condition.

> \* It’s not quite the same, because `compare_exchange` can suffer from ABA
> problems in which it will see a later value in the modification order that
> just happened to be same and succeed. However, in this code values can never
> be reused so we don’t have to worry about that.

In our case, we can simply replace the store with a compare exchange of the old
value and itself plus one (returning `None` instead if the addition overflowed,
to prevent overflowing the atomic). Should the `compare_exchange` fail, we know
that some other thread inserted a value in the modification order after the
value we loaded. This isn’t really a problem — we can just try again and again
until we succeed, and `compare_exchange` is even nice enough to give us the
updated value so we don’t have to load again. Also note that after we’ve updated
our value of the atomic, we’re guaranteed to never see the old value again, by
the arrow rules from the previous chapter.

So here’s how it looks with these changes appplied:

```rust
# use std::sync::atomic::{self, AtomicU64};
static COUNTER: AtomicU64 = AtomicU64::new(0);
pub fn get_id() -> Option<u64> {
    // Load the counter’s initial value from some place in the modification
    // order (it doesn’t matter where, because the compare exchange makes sure
    // that our new value appears directly after it).
    let mut value = COUNTER.load(atomic::Ordering::Relaxed);
    loop {
        // Attempt to add one to the atomic.
        let res = COUNTER.compare_exchange(
            value,
            value.checked_add(1)?,
            atomic::Ordering::Relaxed,
            atomic::Ordering::Relaxed,
        );
        // Check what happened…
        match res {
            // If there was no value in between the value we loaded and our
            // newly written value in the modification order, the compare
            // exchange suceeded and so we are done.
            Ok(_) => break,

            // Otherwise, there was a value in between and so we need to retry
            // the addition and continue looping.
            Err(updated_value) => value = updated_value,
        }
    }
    Some(value)
}
```

This `compare_exchange` loop enables the algorithm to succeed even under
contention; it will simply try again (and again and again). In the below
execution, Thread 1 gets raced to storing its value of `1` to the counter, but
that’s okay because it will just add `1` to the `1`, making `2`, and retry the
compare exchange with that, eventually resulting in a unique ID.

```text
Thread 1   COUNTER   Thread 2
╭───────╮   ┌───┐   ╭───────╮
│ load  ├───┤ 0 ├───┤ load  │
╰───╥───╯   └───┘   ╰───╥───╯
╭───⇓───╮   ┌───┬─┐ ╭───⇓───╮
│  cas  ├───┤ 1 │ └─┤  cas  │
╰───╥───╯   └───┘   ╰───────╯
╭───⇓───╮ ┌─┬───┐
│  cas  ├─┘ │ 2 │
╰───────╯   └───┘
```

> `compare_exchange` is abbreviated to CAS here (which stands for
> compare-and-swap), since that is the more general name for the operation. It
> is not to be confused with `compare_and_swap`, a deprecated method on Rust
> atomics that performs the same task as `compare_exchange` but has an inferior
> design in some ways.

There are two additional improvements we can make here. First, because our
algorithm occurs in a loop, it is actually perfectly fine for the CAS to fail
even when there wasn’t a value inserted in the modification order in between,
since we’ll just run it again. This allows to switch out our call to
`compare_exchange` with a call to the weaker `compare_exchange_weak`, that
unlike the former function is allowed to _spuriously_ (i.e. randomly, from the
programmer’s perspective) fail. This often results in better performance on
architectures like ARM, since their `compare_exchange` is really just a loop
around the underlying `compare_exchange_weak`. x86\_64 however will see no
difference in performance.

The second improvement is that this pattern is so common that the standard
library even provides a helper function for it, called `fetch_update`. It
implements the boilerplate `load`-`loop`-`match` parts for us, so all we have to
do is provide the closure that calls `checked_add(1)` and it will all just work.
This leads us to our final code for this example:

```rust
# use std::sync::atomic::{self, AtomicU64};
static COUNTER: AtomicU64 = AtomicU64::new(0);
pub fn get_id() -> Option<u64> {
    COUNTER.fetch_update(
        atomic::Ordering::Relaxed,
        atomic::Ordering::Relaxed,
        |value| value.checked_add(1),
    )
    .ok()
}
```

These CAS loops are the absolute bread and butter of concurrent programming;
they’re absolutely everywhere and essential to know about. Every other RMW
operation on atomics can (and often is, if the hardware doesn’t have a more
efficient implementation) be implemented via a CAS loop. This is why CAS is seen
as the canonical example of an RMW — it’s pretty much the most fundamental
operation you can get on atomics.

I’d also like to briefly bring attention to the atomic orderings used in this
section. They were mostly glossed over, but we were exclusively using `Relaxed`,
and that’s because for something as simple as a global ID counter, _you never
need more than `Relaxed`_. The more complex cases which we’ll look at later
definitely do need stronger orderings, but as a general rule, if:

- you only have one atomic, and
- you have no other related pieces of data

`Relaxed` is more than sufficient.

## “Out-of-thin-air” values

One peculiar consequence of the semantics of `Relaxed` operations is that it is
theoretically possible for values to come into existence “out-of-thin-air”
(commonly abbreviated to OOTA) — that is, a value could appear despite not ever
being calculated anywhere in code. In particular, consider this setup:

```rust
# use std::sync::atomic::{self, AtomicU32};
let x = AtomicU32::new(0);
let y = AtomicU32::new(0);

// Thread 1:
let r1 = y.load(atomic::Ordering::Relaxed);
x.store(r1, atomic::Ordering::Relaxed);

// Thread 2:
let r2 = x.load(atomic::Ordering::Relaxed);
y.store(r2, atomic::Ordering::Relaxed);
```

When starting to draw a diagram for a possible execution of this program, we
have to first lay out the basic facts that we know:
- `x` and `y` both start out as zero
- Thread 1 performs a load of `y` followed by a store of `x`
- Thread 2 performs a load of `x` followed by a store of `y`
- Each of `x` and `y` take on exactly two values in their lifetime

Then we can start to construct boxes:

```text
Thread 1      x       y      Thread 2
╭───────╮   ┌───┐   ┌───┐   ╭───────╮
│  load ├─┐ │ 0 │   │ 0 │ ┌─┤ load  │
╰───╥───╯ │ └───┘   └───┘ │ ╰───╥───╯
    ║     │   ?───────────┘     ║
╭───⇓───╮ └───────────?     ╭───⇓───╮
│ store ├───┬───┐   ┌───┬───┤ store │
╰───────╯   │ ? │   │ ? │   ╰───────╯
            └───┘   └───┘
```

At this point, if either of those lines were to connect to the higher box then
the execution would be simple: that thread would forward the value to its lower
box, which the other thread would then either read, or load the same value
(zero) from the box above it, and we’d end up with zero in both atomics. But
what if they were to connect downwards? Then we’d end up with an execution that
looks like this:

```text
Thread 1      x       y      Thread 2
╭───────╮   ┌───┐   ┌───┐   ╭───────╮
│  load ├─┐ │ 0 │   │ 0 │ ┌─┤ load  │
╰───╥───╯ │ └───┘   └───┘ │ ╰───╥───╯
    ║     │   ┌───────────┘     ║
╭───⇓───╮ └───┼───────┐     ╭───⇓───╮
│ store ├───┬─┴─┐   ┌─┴─┬───┤ store │
╰───────╯   │ ? │   │ ? │   ╰───────╯
            └───┘   └───┘
```

But hang on — it’s not fully resolved yet, we still haven’t put in a value in
those lower question marks. So what value should it be? Well, the second value
of `x` is just copied from from the second value of `y`, so we just have to find
the value of that — but the second value of `y` is itself copied from the second
value of `x`! This means that we can actually put any value we like in that box,
including `0` or `42`, and the logic will check out perfectly fine — meaning if
this program were to execute in this fashion, it would end up reading a value
produced out of thin air!

Now, if we were to strictly follow the rules we’ve laid out thus far, then this
would be totally valid thing to happen. But luckily, the authors of the C++
specification have recognized this as a problem, and as such refined the
semantics of `Relaxed` to implement a thorough, logically sound, mathematically
proven formal model that prevents it, that’s just too complex and technical to
explain here—

> No “out-of-thin-air” values can be computed that circularly depend on their
> own computations.

Just kidding. Turns out, it’s a *really* difficult problem to solve, and to my
knowledge even now there is no known formal way to express how to prevent it. So
in the specification they just kind of hand-wave and say that it shouldn’t
happen, and that the above program must always give zero in both atomics,
despite the theoretical execution that could result in something else. Well, it
generally works in practice so I can’t complain — it’s just a very interesting
detail to know about.
