+++
title = "sync.Map, and the Question I Should've Asked First"
date = 2026-09-04
description = "Why not just guard a plain map with a mutex? Go's own docs mostly agree with that."
[taxonomies]
tags = ["go", "concurrency"]
+++

For a long time I just ignored `sync.Map` and kept writing a plain `map` behind a `sync.Mutex`, because that shape I already trust, and `sync.Map` sat in the back of my head as "the concurrent map I'm apparently not using". When I finally looked, the standard library's own docs say most code shouldn't use it either.

## why you need something

A regular map with a concurrent read and write doesn't just give wrong answers, the runtime detects it and kills the process.

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

It's a fatal error and not a panic, so `recover` can't catch it, and ranging over the map while another goroutine writes triggers it too. So something has to guard it.

## what the docs say

`sync.Map` describes itself as safe for concurrent use "without additional locking or coordination", and using it looks like a normal map with `Store`, `Load` and `Delete`. Then the next part of the same doc comment says most code should use a plain map with separate locking, for better type safety, and that `sync.Map` is for two cases.

The first is a key written once and then read a lot, like a cache that only grows or config loaded at startup. The second is goroutines working on disjoint sets of keys, A only touches `a1`, `a2`, B only touches `b1`, `b2`, so they share one map but never fight over the same entry.

## why those two

The doc doesn't explain it, but the old implementation does. Up to Go 1.23 `sync.Map` had two maps inside, a read-only one you could load from without any lock, and a dirty one behind a mutex for new keys. A lookup that missed the read map took the lock and checked the dirty map, and after enough misses the dirty map got promoted to be the new read map.

With that in mind the two cases make sense. A write-once key goes through the slow locked path one time, then every later read is lock-free. Disjoint keys mean goroutines aren't contending anyway. Anything else, and you're paying for two maps and a promotion scheme where one mutex would have done the same job.

Go 1.24 replaced all this with a hash-trie, partly to get rid of the warm-up where a key needed a bunch of misses before its reads became lock-free. There was an opt-out for the old version at first, and I think it's gone now. So the two-map design is history, but it's still why the docs list those two cases.

## so, the mutex

For almost everything I'd want a concurrent map for, mutex plus map is better. `sync.Map` takes and returns `any`, so you type-assert on every access, there's no `Len()`, and `Range` doesn't lock the whole map, so if someone modifies it while you range you don't get a consistent snapshot. If I can't say ahead of time that my access pattern is one of the two cases, I'm not who it was built for, and I think picking it because it has "sync" in the name is the exact mistake the doc comment is warning about.

## notes

- `Load`, `Store`, `Delete` are amortized constant time, single calls can be slower.
- Don't copy a `sync.Map` after first use, same rule as every other sync type.
- The zero value just works, which is nicer than remembering to init a map and a mutex.
- `LoadOrStore`, `LoadAndDelete`, `Swap`, `CompareAndSwap`, `CompareAndDelete` are the one real win, they're the operations you'd otherwise need to hold your own lock across two map calls for.
- `Swap` and the `CompareAnd*` ones came in Go 1.20, `Clear` in 1.23.
