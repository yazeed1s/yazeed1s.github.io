+++
title = "The Three Queues Hiding Inside a Channel"
date = 2026-08-12
description = "The mental model I use for Go channels, a buffer, two queues of waiting goroutines, and a lock, plus the gotchas that come out of it."
[taxonomies]
tags = ["go", "concurrency", "systems programming"]
+++

Everyone learns channels with the Rob Pike line, don't communicate by sharing memory, share memory by communicating, and it's a nice sentence that tells you nothing about what happens when two goroutines hit the same channel. What made channels click for me was reading what the runtime actually allocates, because most of the weird behavior comes straight out of that.

## what's inside

A channel is a pointer to a struct the runtime calls `hchan`. It holds a circular buffer sized to the capacity you gave `make` (no buffer at all if you gave 0, that's what unbuffered means), a queue of goroutines blocked on receive, a queue of goroutines blocked on send, and one mutex over all of it. Every send, receive and close takes that lock, looks at the buffer and the two queues, does one of a few things, and unlocks.

If the buffer has room, a send drops the value in and leaves. If a receiver shows up and a sender is already waiting, the value goes from the sender directly to the receiver. That direct handoff is the only way an unbuffered channel works at all, a send can't finish until a receiver is right there, so I stopped thinking of unbuffered as "capacity 0" and more as a different mode, a rendezvous, which you use when you care that two things happened together and not just that data moved.

One small thing I liked, for a buffered channel at least one of the two wait queues is always empty. If a receiver were waiting, the runtime would have handed it the value instead of leaving it in the buffer, and the same goes the other way for senders.

## close

Closing wakes every blocked receiver with the zero value and `ok == false`, and every receive after that returns the same thing forever. That's why `for v := range ch` ends cleanly. Values already in the buffer still come out first with `ok == true`, close doesn't throw them away.

Sending on a closed channel panics, always, and closing twice panics too. So for a receiver close means "I'll keep answering", and for a sender it means "you're not allowed here anymore", which is why only the sender should close, a receiver closing it would make some unrelated sender panic.

## nil channels

A channel that was never `make`'d blocks forever on send and on receive, but `close(nil)` panics, which I keep forgetting. Forget to `make` one and your program just freezes, and the stack trace says `chan receive (nil chan)`, which is fair.

It's useful on purpose inside `select` though. A nil case is never ready, so setting `ch = nil` after a channel is done turns that branch off while the others keep working. Kinda elegant, blocking forever as a feature.

## select

If several cases are ready, `select` picks one at random, not the first one. I'd guess that's deliberate so nobody depends on case order, but it means you can't use order as priority, if you need priority you check the important channel in its own non-blocking select first.

The one that actually got me is that `select` evaluates every channel expression and every value being sent, for all cases, before it picks. So `case out <- compute():` runs `compute()` even if another case wins. The body of a losing branch never runs, but its expressions do, so side effects in case expressions are a bad idea.

Mix that with the close rules and you get this:

```go
c := make(chan struct{})
close(c)
select {
case c <- struct{}{}:
    // panics if picked
case <-c:
    // fine if picked
}
```

Receiving from a closed channel never blocks, and sending to a closed one doesn't block either, it panics, so both cases count as ready and `select` flips a coin. The program panics about half the time with no data race anywhere. I didn't believe it until I ran it a bunch of times.

Under the hood `select` also sorts the channels it touches and locks them in that order, so two selects on the same channels in different order can't deadlock each other. Branch choice is random but lock order isn't.

## ownership

The part that changed how I write Go isn't the mechanics. Sending a pointer, slice or map on a channel is really handing over ownership, and nothing in the language enforces it, both sides can still touch the data after. The rule "don't touch it after you send it" is doing the work a mutex would do, and it's the same trust as `free()` in C. Channels handle the synchronization for you, but you still have to think about who owns what.

And the value is copied, twice if it goes through the buffer (sender to slot, slot to receiver). For big structs send a pointer. The compiler even refuses element types of 64KB or more, not that you'd want to get close.

## notes

- Two channel values are equal only if they're the same channel, assigning one just gives you another handle.
- `len` and `cap` on a nil channel return 0, no panic.
- A goroutine blocked on a channel nobody will ever touch again is never collected, that's a real leak.
- `chan T` converts to `chan<- T` or `<-chan T`, never back.
- `select {}` blocks forever on purpose, sometimes used to keep `main` alive.
