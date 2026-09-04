+++
title = "The Three Queues Hiding Inside a Channel"
date = 2026-09-04
description = "The mental model I actually use for Go channels, not a pipe but three queues and a lock, plus the gotchas that fall out of it."
[taxonomies]
tags = ["go", "concurrency", "systems programming"]
+++

Everyone gets taught channels the Rob Pike way, don't communicate by sharing memory, share memory by communicating, and it's a nice sentence but it doesn't tell you anything about how the thing actually behaves once two goroutines are fighting over it. The model that finally made channels click for me wasn't the CSP quote, it was picturing what's actually sitting inside a channel value at runtime. Almost every weird behavior channels have falls out of that picture once you hold it in your head, and once I actually went and read how the runtime implements this, a bunch of stuff that felt like magic turned out to be pretty mechanical.

## what a channel actually is

A channel value is a pointer to a small struct the runtime calls `hchan`. Making one with `make(chan int, 3)` allocates that struct, it doesn't hand you a pipe or a socket or anything like that, it's just memory with a lock on it. Inside that struct there are three things that matter.

There's a circular buffer of values sized to whatever capacity you passed to `make` (zero if you left it out, that's what "unbuffered" means, there's no array at all), a queue of goroutines currently blocked trying to receive from this channel, a queue of goroutines currently blocked trying to send, and one mutex guarding all three that every single send, receive, and close operation has to acquire before it's allowed to touch anything.

So `ch <- x` isn't "push to a queue" in the abstract sense, it's lock the struct, look at the buffer and the two goroutine queues, do whichever of a few things applies, then unlock. Same for receiving. The rest of this post is just walking through what those few things are, because once you know them the gotchas stop being gotchas.

## the buffer is only ever half the story

A buffered channel is not "an array with a lock", even though that's basically what people picture. The buffer is the middle piece, and the two goroutine queues only get used when the buffer can't absorb the operation right away.

If the buffer has room, a send just drops the value in and moves on, no goroutine queue touched at all. If the buffer is empty and a receiver shows up while a sender is already blocked waiting, the receiver takes straight from the sender's hand, the value never touches the buffer.

That direct handoff path is the whole reason unbuffered channels work at all. There's no buffer to hold anything, so a send on an unbuffered channel literally cannot complete until a receiver is standing there ready to take it, which is why people call it a rendezvous instead of a queue. I used to think of unbuffered as "buffered with capacity 0", a boring edge case, but it's really a different mode, synchronous handoff instead of a mailbox, and the two get reached for in real code for different reasons, one for passing data, one for making sure two things happened at the same moment.

A small invariant falls out of this and I like it. If the buffer isn't empty, the receiving queue has to be empty too, and if the buffer isn't full, the sending queue has to be empty too, so for a buffered channel at least one of the two goroutine queues is always empty. Makes sense once you say it out loud, if a receiver were already blocked and waiting, why would the runtime leave a value sitting in the buffer instead of handing it straight over, but it's not something I'd have guessed just from using channels day to day.

## closing does something I didn't expect for a while

Close a channel and every blocked receiver on it wakes up immediately with a zero value, not an error, a zero value, plus a second bool that's `false` telling you nothing was actually sent. That's not a one-time event either, receive from a closed, drained channel again and you get the same thing, forever, `0, false`, `0, false`, as many times as you ask.

A closed channel never blocks on receive again, period. That's the whole trick behind `for v := range ch` terminating cleanly, the loop just keeps asking and Go keeps saying "closed, nothing left" until the range sees the false and stops.

What I didn't expect the first time I read this carefully is that sending on a closed channel is the opposite of gentle. It panics right away instead of handing back a zero value and a bool, and it's not a "maybe" panic depending on timing either, closed-then-send is always a panic.

So the two ends of a channel disagree completely about what closing means. For a receiver it means "keep asking, I'll always answer." For a sender it means "you're not allowed here anymore." That asymmetry is the entire reason the convention is only the sender closes, never the receiver, because if a receiver closed a channel a sender didn't know about, the very next send from that sender panics for no reason it could've predicted. Even closing twice panics, close is a one-way door and Go really does not let you pretend otherwise.

Values already sitting in the buffer before close was called don't disappear either, they're still there and still get handed out with `ok = true` before the channel switches over to infinite-zero-values mode. Close doesn't flush the buffer, it just stops accepting new sends and promises receivers will never block again.

## nil channels are the trap door

A nil channel, one that's never been through `make`, blocks forever on send and forever on receive, both directions, no exception. It doesn't panic on those, it just hangs the goroutine there permanently, but `close(nilChan)` does panic, which always trips me up because it's the opposite of what send and receive do on the same nil value.

This looks useless until you see it used on purpose inside a `select`. A nil case in a select is never the one that's ready, so assigning `ch = nil` after you're done with it is a clean way to turn off one branch of a select without ripping the whole select apart, the other cases keep working and that one silently never fires again. Kind of elegant honestly, using "blocks forever" as a feature instead of a bug.

The first time you hit a real nil-channel deadlock by accident though, declared a channel variable and forgot to `make` it, it just looks like your program froze for no reason. The stack trace says "chan receive (nil chan)" which, fair, that's exactly what's happening, you just didn't mean for it to.

## select doesn't pick the first ready case

If two or more cases in a `select` are ready at the same instant, Go does not take them top to bottom, it picks uniformly at random among the ready ones. That surprised me more than it probably should have.

I'd guess the randomness is deliberate, so people don't write code that quietly depends on case ordering and then have it break the moment someone reorders the cases. But it means you cannot use case order as a priority mechanism, which I definitely assumed you could before actually checking. If you need real priority between channels you have to nest selects or check one channel with its own non-blocking select first, the outer select's ordering buys you nothing.

The one that actually got me took reading the runtime steps to believe it. A `select` evaluates every channel expression and every value-to-send expression for every case, top to bottom, before it decides which branch to run, not just the winning one, all of them. So if one of your case statements is `case resultCh <- computeExpensiveThing():`, that function call runs whether or not that branch ends up being the one selected, as long as select even considers it. I'd always assumed a losing select branch never executes any of its own code, and that's true for the branch body, but the send value and the channel itself get evaluated regardless. Side effects in a select case expression are basically always a mistake, and I hadn't thought about why until now.

## the coin flip inside select

A nasty little example falls straight out of combining the closed-channel rules with the random pick, put both a send case and a receive case on the same already-closed channel in one select.

```go
c := make(chan struct{})
close(c)
select {
case c <- struct{}{}:
    // panics if this one gets picked
case <-c:
    // fine if this one gets picked
}
```

Receiving from a closed channel never blocks, so that case is always ready. Sending to a closed channel also never blocks, it just panics the instant it runs, so from select's point of view that case counts as "ready" too, non-blocking counts as ready even when what's waiting on the other side of ready is a panic.

Both cases are non-blocking, so select picks between them uniformly at random, meaning this program panics about half the time and runs clean the other half. Same code, same input, no data race even, just a coin flip baked into the language runtime. First time I saw that I did not believe it until I ran it a few dozen times myself.

There's also a boring but useful implementation detail underneath all this. Before picking anything, select sorts all the channels involved in its cases and locks them in that order, specifically to avoid deadlocking against some other goroutine locking the same two channels in a different order at the same time. I don't think about this day to day, but it's nice to know the randomness and the locking aren't fighting each other, the runtime sorts first so lock order is deterministic even though branch order isn't.

## ownership, not just values

The part of all this that actually changed how I write Go isn't the mechanics though, it's a framing thing. Sending a value on a channel is, in practice, handing off ownership of whatever that value points to, not just copying some bytes to another goroutine.

The language doesn't enforce this, nothing stops both sides from touching a shared slice after the handoff. But the convention, don't touch it after you've sent it, is doing the real work that a mutex would otherwise have to do explicitly. It's the same trust-based discipline C asks of you with `free()`, just moved into a different shape. I don't think that gets said enough when channels are pitched as the safe concurrent option, they're safer in the sense that the synchronization is handled for you, not in the sense that you can stop thinking about who owns what.

Something related that's easy to miss, the value itself gets copied, not moved, and if it passes through the buffer it actually gets copied twice, once from sender into the buffer slot, once from the buffer slot into the receiver. For small values that's nothing, but I've seen people channel around large structs by value without thinking about it. The compiler will even refuse element types past a certain size, 65536 bytes on the standard compiler, not that you'd want to get anywhere near that. Past a certain size you want a pointer in the channel, not the struct itself, same reasoning as passing large structs as function arguments.

## notes

- two channel values compare equal only if they're literally the same underlying channel, assigning a channel variable to another just gives you two handles to the same struct, not a copy of anything
- `cap(ch)` and `len(ch)` both return 0 on a nil channel instead of panicking, small thing but nice not to have to nil-check before calling them
- a goroutine parked in a channel's wait queue can't be garbage collected even if nothing else references it, so a goroutine leak from a channel nobody's writing to anymore is a real, silent memory leak, not just a stuck goroutine you can ignore
- a bidirectional `chan T` converts implicitly to `chan<- T` or `<-chan T`, but you can't go the other way, and you definitely can't convert send-only to receive-only or back, the compiler just refuses
- `select{}` with zero cases blocks the current goroutine forever, on purpose, I've seen it used as a deliberate "keep main alive" idiom and it reads weird until you know that's what it's doing
