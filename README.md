# cache

A general-purpose cache for [Ecko](https://ecko.sh), written in Ecko. An
in-memory LRU with TTL sitting in front of an optional disk store - so a value
can be memoized for the life of a process, or across separate `ecko` runs.

No native code: it composes `std.fs`, `std.time`, `std.hash`, and `std.json`
over a `cell`. This is the general cache; the `ai` prompt cache
(`ECKO_AI_CACHE`) is separate and specific to LLM calls.

## Install

```bash
ecko add https://github.com/ecko-sh/cache
```

`ecko add` vendors the package into `./vendor/cache/` and pins it by SHA-256 in
`ecko.lock`. Grant it filesystem access for the disk tier in your `ecko.json`:

```json
{
  "dependencies": {
    "cache": {
      "source": "https://github.com/ecko-sh/cache",
      "grant": ["fs:read", "fs:write"]
    }
  }
}
```

A memory-only cache (`open("")`) uses no filesystem and needs no grant.

## Usage

```ecko
import cache

c = cache.open(".ecko-cache")            # disk-backed; survives across runs

# Memoize an expensive fetch for 5 minutes. On the next run within the TTL,
# this returns the cached value without calling the function again.
board = cache.remember(c, "board", 300, || json.decode(http.get(url).body))

cache.set(c, "greeting", "hello", 60)    # ttl seconds; 0 = never expires
cache.get(c, "greeting")                 # "hello", or null on a miss/expiry
cache.has(c, "greeting")                 # true (non-mutating check)
cache.delete(c, "greeting")
cache.clear(c)                           # drop everything this cache stored
```

## API

Every function takes the handle from `open` as its first argument.

| Function | Description |
|---|---|
| `open(dir, opts?)` | Open a cache. `dir` is a path (disk-backed) or `""`/`null` (memory-only). `opts.max` is the memory LRU capacity (default `1000`). Returns a handle. |
| `get(c, key)` | The value for `key`, or `null` on a miss or expiry. |
| `set(c, key, value, ttl?)` | Store `value` under `key`. `ttl` is in **seconds**; `0` (default) never expires. Returns `value`. |
| `remember(c, key, ttl, fn)` | Return the cached value, or run `fn()` once, cache its result with `ttl`, and return it. |
| `has(c, key)` | `true` if `key` is present and unexpired. Non-mutating. |
| `delete(c, key)` | Remove `key` from memory and disk. |
| `clear(c)` | Empty the cache (removes only the files this cache wrote). |

**Values** must be JSON-serializable when the cache is disk-backed: null,
bool, int, float, string, list, map, struct. Functions and cells do not
persist. A memory-only cache can hold any value.

**`remember` and null.** A cached `null` is indistinguishable from a miss, so
`remember` will re-run its function. Use `set`/`get` if you need to cache
`null` explicitly.

## How it works

- **Two tiers.** `get` checks the in-memory LRU first, then disk; `set` writes
  both. The memory tier is bounded by `max` and evicts least-recently-used
  entries (they remain on disk until their TTL expires). The handle carries the
  memory tier in a `cell`, so sharing it into `pmap`/spawned tasks shares one
  tier safely.
- **Disk layout.** Each entry is one JSON file named by the SHA-256 of its key,
  holding `{"expires": <ms, 0 = never>, "value": <any JSON>}`. Writes are
  atomic (write to a `.tmp` file, then rename over the target), so a concurrent
  reader never sees a half-written entry. A missing or corrupt file reads as a
  miss and never raises.
- **TTL.** Expiry is stored as an absolute wall-clock time (Unix ms), so it is
  honored across process runs. Expired entries are dropped on read.

## Testing

```bash
ecko test tests/
```

The tests are offline and deterministic - they cover round-trips, type
preservation, TTL expiry, disk persistence across handles, LRU eviction,
memoization, and `clear` scoping. `example.ecko` is a runnable
memoize-across-runs demo.

## License

MIT — see [LICENSE](LICENSE).
