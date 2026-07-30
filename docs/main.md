# cache

## `open(dir, opts = empty_map())`

open(dir, opts?) -> a cache handle.

`dir` empty or null makes the cache memory-only; a path makes it disk-backed,
so entries survive across process runs. `opts.max` caps the in-memory tier
(default 1000) and eviction is least-recently-used - an evicted entry is gone
from memory but still on disk.

```ecko
c = cache.open(".ecko-cache", { max: 500 })
mem = cache.open("")               # memory-only
```

## `set(c, key, value, ttl = 0)`

set(c, key, value, ttl?) -> the value, so it can be used inline.

`ttl` is in seconds; 0 (the default) never expires. Writes both tiers.

```ecko
cache.set(c, "board", data, 300)   # five minutes
```

## `get(c, key)`

get(c, key) -> the value, or null when there is nothing live for `key`.

Memory first, then disk. An expired entry is deleted and reads as a miss, and
a disk hit is promoted back into memory. A hit bumps recency.

## `has(c, key)`

has(c, key) -> whether a live entry exists for `key`.

Non-mutating, which is the difference from `get`: it neither bumps recency
nor promotes a disk entry into memory, so probing cannot evict anything.

## `remember(c, key, ttl, producer)`

remember(c, key, ttl, producer) -> the cached value, computing it if absent.

On a hit the cached value is returned and `producer` is never called. On a
miss `producer()` runs once, its result is stored with `ttl`, and returned.

A cached null reads as a miss, so `producer` would run again - use `set` and
`get` directly if null is a value you need to cache.

```ecko
board = cache.remember(c, "board", 300, fn() http.get(url).body)
```

## `delete(c, key)`

delete(c, key) -> null. Removes `key` from both tiers.

## `clear(c)`

clear(c) -> null. Empties the memory tier and removes this cache's own disk
files, leaving anything else in the directory alone.
