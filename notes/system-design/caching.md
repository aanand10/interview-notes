# Caching strategies

Caches trade **freshness** for **speed** and **reduced load**.

## Read patterns

### Cache-aside (lazy loading)

The app checks the cache first; on a miss it reads the DB and writes to the cache.

- ✅ Only caches what is actually requested
- ❌ First request is always slow; risk of stale data

### Read-through

The cache itself loads from the DB on a miss. Simpler app code, same staleness trade-off.

## Write patterns

### Write-through

Write to cache and DB together. Consistent, but every write pays both costs.

### Write-back (write-behind)

Write to cache, flush to DB asynchronously. Fast writes, but you can lose data if the cache dies.

## Eviction

- **LRU**: evict least recently used (most common)
- **LFU**: evict least frequently used
- **TTL**: expire after a fixed time

## Common problems

- **Thundering herd / cache stampede**: many requests miss at once. Fix with request coalescing or locks.
- **Invalidation**: "There are only two hard things in Computer Science: cache invalidation and naming things."
