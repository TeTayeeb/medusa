# Software Design Document — @medusajs/caching-redis

## 1. Purpose

Provide a production-grade Redis-backed cache for Medusa's Caching Module. Beyond simple key-value caching, the provider implements **tag-based invalidation** (allowing grouped cache clearing), **data compression**, and **graceful degradation** on Redis unavailability.

## 2. Architecture

```
Modules.CACHING
  └── ModuleProvider (caching-redis)
        ├── Loaders
        │     ├── connection.js  ← ioredis client + DI registration
        │     └── hash.js        ← (hashing utilities)
        └── RedisCachingProvider
              ├── get({ key, tags })
              ├── set({ key, data, ttl, tags, options })
              └── invalidate(keyOrTag)
```

## 3. Redis key schema

The provider manages several Redis key types:

| Key pattern | Type | Purpose |
|---|---|---|
| `<prefix><key>` | Hash | Cache entry: `data` (compressed), `options` |
| `<prefix><key>:tags` | String | Serialized tag ID array for a cache entry |
| `<prefix>tag:<hash>` | Set | Member cache keys for a given tag |
| `<prefix>tag-dict` | Hash | Tag name → numeric ID dictionary |
| `<prefix>tag-next-id` | String | Auto-increment counter for tag IDs |
| `<prefix>tag-refcount` | Hash | Reference counts per tag ID |
| `<prefix>tag-rev-dict` | Hash | Numeric ID → tag name reverse lookup |

Tag **interning** (name → numeric ID) reduces Redis memory usage for entries with many shared tags.

## 4. `get` operation

```
get({ key, tags }):
  if key:
    keyName = prefix + key
    buffer = redisClient.hgetBuffer(keyName, "data")
    if buffer null → return null (cache miss)
    return decompress(buffer)
    catch ConnectionError → warn + return null

  if tags:
    for each tag:
      tagKey = getTagKey(tag)
      members = redisClient.smembers(tagKey)
      use pipeline to hgetBuffer all member "data" fields
      decompress each → collect non-null results
    return results
    catch ConnectionError → warn + return []
```

## 5. `set` operation

```
set({ key, data, ttl, tags, options }):
  keyName = prefix + key
  effectiveTTL = ttl ?? defaultTTL
  serialized = JSON.stringify(data)
  compressed = compress(serialized)

  tagIds = internTags(tags)   // tag names → numeric IDs

  pipeline:
    HSETNX keyName "data" compressed
    if options: HSET keyName "options" JSON(options)
    EXPIRE keyName effectiveTTL
    if tags:
      SETNX/SET <key>:tags buffer(tagIds)  // with TTL + 60s padding
      for each (tag, tagId):
        SADD tag:<hash> keyName
        EXPIRE tag:<hash> effectiveTTL + 60s (if TTL present)
  pipeline.exec()
  catch ConnectionError → warn (non-fatal)
```

## 6. `invalidate` operation

Two invalidation paths:

### By key:
```
1. keyName = prefix + key
2. pipeline: GET <key>:tags → parse tagIds
3. DEL keyName, DEL <key>:tags
4. decrementTagRefs(tagIds)  // remove from tag sets; cleanup orphan tag entries
```

### By tag:
```
1. tagKey = getTagKey(tag)
2. SMEMBERS tagKey → list of all cache keys under this tag
3. For each key:
   a. Check options field for autoInvalidate (default: true)
   b. If autoInvalidate: DEL key + DEL key:tags + decrementTagRefs
4. DEL tagKey
```

`decrementTagRefs` decrements the reference count for each tag ID. When ref count reaches zero, the tag dictionary entries are cleaned up.

## 7. Data compression

The provider compresses serialized cache values before storage and decompresses on retrieval. This reduces Redis memory usage for large payloads (product lists, query results) at the cost of CPU cycles. The compression implementation is encapsulated in private methods `#compressData` and `#decompressData`.

## 8. Pipeline usage

Redis pipelines batch multiple commands into a single network round-trip. The provider uses pipelines in both `set` and `invalidate` to ensure atomicity and reduce latency. `pipeline.exec()` returns results for each command in order.

## 9. Graceful degradation

```ts
try {
  // Redis operation
} catch (error) {
  this.logger.warn("Redis connection error during get, returning null")
  return null  // or []
}
```

Cache misses on connection errors allow the application to fall back to its primary data source (database). This is the correct behaviour: caching is a performance optimisation, not a system of record.

## 10. Connection loader

```
connection.js:
  1. Parse redisUrl from options
  2. Create ioredis client with conservative timeouts:
     connectTimeout: 10000, commandTimeout: 5000, lazyConnect: true
  3. Attach event listeners (error, connect, ready, close, reconnecting)
  4. PING test → log success/failure (non-fatal)
  5. Register in DI: redisClient, prefix, logger
```

## 11. `autoInvalidate` option

Each cache entry can opt out of tag-based invalidation:
```ts
options: { autoInvalidate: false }
```
When `false`, the entry is not deleted when its tag is invalidated. Useful for entries that should outlive tag invalidation (e.g. configuration data).

Default: `true` (or absent → treated as `true`).

## 12. Dependencies

| Package | Purpose |
|---|---|
| `ioredis` | Redis client with pipeline, pub/sub support |
| `@medusajs/framework` | `ModuleProvider`, DI container |
