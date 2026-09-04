+++
title = "sync.Map, and the Question I Should've Asked First"
date = 2026-09-04
description = "Why not just guard a plain map with a mutex? Go's own docs answer that directly, and the two exceptions they carve out explain the whole design."
[taxonomies]
tags = ["go", "concurrency"]
+++

First time I ran into `sync.Map` I didn't reach for it, I just kept using a plain `map` behind a `sync.Mutex` like always, mostly because that shape is one I already trust and `sync.Map` felt like one more thing to learn for no obvious reason. It sat in the back of my head for a while as "the concurrent map I'm apparently not using," which is an annoying thing to leave unresolved.

What I found is that `sync.Map` isn't a general-purpose replacement for map-plus-mutex at all, it's a narrow optimization for two specific access patterns, and the standard library says so plainly in its own doc comment, not as an implication I'm reading into it.

## what a plain map won't survive

Start with the problem `sync.Map` exists to fix. A regular Go map isn't safe under concurrent access, and it's not a silent kind of unsafe either, the runtime detects it and kills the whole program.

```go
m := make(map[string]int)

go func() {
    for {
        m["thing"] = 1
    }
}()

go func() {
    for {
        fmt.Println(m["thing"])
    }
}()

select {}

// fatal error: concurrent map read and map write
```

That's not a `panic` you can `recover` from, it's a fatal error, the process is going down. And it's not only read-vs-write that trips it, just ranging over a map while another goroutine writes to it triggers the same fatal error. So something has to guard concurrent access, either a mutex you manage yourself or a type built to handle it internally.

## the pitch, and the very next sentence

`sync.Map` bills itself as a map "safe for concurrent use by multiple goroutines without additional locking or coordination." Using it looks exactly like a regular map:

```go
var m sync.Map

m.Store("foo", "bar")
value, ok := m.Load("foo")
fmt.Println(value, ok) // foo true

m.Delete("foo")
value, ok = m.Load("foo")
fmt.Println(value, ok) // <nil> false
```

No mutex in sight, no `recover`, nothing crashes under concurrent load. So far it does look like the drop-in fix. Then the same doc comment turns around and says most code should use a plain map with its own locking instead, for better type safety, right there in the sentence after the pitch. That's the standard library naming its own type and then telling you not to reach for it by default.

## the two shapes it's actually for

The doc doesn't leave "most code" vague, it names exactly two access patterns where `sync.Map` wins.

The first is a cache that only grows. A key gets written once, maybe the first time it's computed, and after that it's read constantly and never written again. A memoized function's results, or a config value loaded at startup and read on every request afterward, both fit this shape.

The second is a map split across goroutines by key rather than by lock. Several goroutines are all touching the same `sync.Map`, but goroutine A only ever reads and writes keys `"a1"`, `"a2"`, `"a3"`, while goroutine B only ever touches `"b1"`, `"b2"`. Nobody's actually contending for the same entry, even though they're technically sharing one map.

Outside those two shapes, a mutex-guarded map is the better call, in the standard library's own words.

## why those two shapes specifically

None of that reasoning is in the doc comment, you have to look at how the type used to be built to see why those two cases are the special ones, and once I did it stopped feeling arbitrary.

For a long time, up through Go 1.23, `sync.Map` kept two maps internally, not one. A read-only map you could load from with no lock at all, and a dirty map behind an actual mutex for anything the read map didn't have yet. A lookup that missed the read map fell back to locking and checking the dirty map, and that counted as a miss. Once enough misses piled up to justify the cost, the dirty map got copied over and promoted into a new read map, and the cycle started again.

Once you see that structure, the two blessed shapes stop being a random list. Write-once-read-forever means a key needs the slow, locked path exactly once, then every read after that hits the lock-free map for good. Disjoint keys means goroutines aren't colliding on the same entries in the first place, so the contention a shared mutex would create barely happens here either. Outside those two shapes you're paying for two maps and a promotion scheme and getting nothing back for it, a single map behind a single mutex does the same job with less machinery.

## the design didn't stay put

What I didn't expect digging into this is that the two-map design had a real cost of its own, one the Go team went back and fixed instead of just documenting around. Go 1.24 replaced the internals with a hash-trie map, specifically to remove what the release notes called ramp-up time, the old design needed a string of misses before a key's reads actually became lock-free, so there was a cold-start tax built into the exact fast path people picked `sync.Map` for in the first place. Go 1.24 shipped it behind a fallback you could opt out of if it broke something, and by Go 1.26 that fallback is gone and the hash-trie is just the implementation, full stop.

So the read-only-plus-dirty-map design in the previous section is history at this point, not current behavior. I still think it's worth knowing, it's the reason the two access patterns in the docs are the ones they are, and that reasoning didn't change just because the data structure underneath did.

## so, back to the mutex

Which loops back to the actual question. For nearly everything I'd reach for a concurrent map to do, a mutex around a plain map wins outright, and the standard library says so plainly, better type safety being the exact phrase it uses. `sync.Map` still takes `any` and returns `any` on every call, so you're type-asserting on every access no matter what Go version you're on, where `map[string]int` guarded by a mutex just never makes you do that. It also has no `Len()`, and `Range` doesn't lock the whole map for the duration, so a range over a `sync.Map` being modified concurrently isn't guaranteed to see a consistent snapshot, which is a real cost if you need one.

`sync.Map` only pays off once you can point at your access pattern and say it's one of the two specific shapes ahead of time. If you can't, you're not who it was built for, and reaching for it because the name starts with "sync" and sounds like the safe default is exactly the assumption the doc comment is trying to talk you out of.

## notes

- `Load`, `Store`, and `Delete` are documented as amortized constant time, amortized is doing real work in that sentence, individual calls can be slower, the promotion/rebalancing underneath is what makes the average hold
- like every other sync primitive, a `sync.Map` must not be copied after it's been used, copying it copies the internal state and the safety guarantee just quietly stops applying
- the zero value works with no setup at all, that part is genuinely nicer than mutex-plus-map, where you have to remember to initialize both pieces yourself
- `LoadOrStore`, `LoadAndDelete`, `Swap`, `CompareAndSwap`, and `CompareAndDelete` exist because those are exactly the operations that would otherwise force you to hold your own mutex across two separate map operations, that's the one place `sync.Map` hands you something rolling your own doesn't give for free
- `CompareAndSwap`, `CompareAndDelete`, and `Swap` only landed in Go 1.20, `Clear` in Go 1.23, so a lot of code written before that had to fake these with `LoadOrStore` and a loop
