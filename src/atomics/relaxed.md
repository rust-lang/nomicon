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
data;
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

Let’s try to figure out where the line in Thread 2’s access joins up. The rules
from before don’t help us much unfortunately since there are no arrows
connecting that operation to anything, so we can’t immediately rule anything
out. As a result, we end up facing a situation we haven’t faced before: there is
_more than one_ potential value for Thread 2 to read.

And this is where we encounter the big limitation with unsynchronized data
accesses: the price we pay for their speed and optimization capability is that
this situation is considered **Undefined Behavior**. For an unsynchronized read
to be acceptable, there has to be _exactly one_ potential value for it to read,
and when there are multiple like in this situation it is considered a data race.

## “Out-of-thin-air” values
