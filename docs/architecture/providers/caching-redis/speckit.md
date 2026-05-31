# SpecKit — @medusajs/caching-redis

---

## 1. Unit specs — `get` (single key)

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U1 | Cache hit | Key exists in Redis | Returns decompressed, deserialized value |
| U2 | Cache miss | Key not in Redis | Returns `null` |
| U3 | Redis connection error | Redis unavailable | Returns `null`; logs warning; does not throw |
| U4 | Prefix applied | `key: "product:1"`, `prefix: "mc:"` | Redis key looked up as `"mc:product:1"` |
| U5 | Data decompressed | Stored as compressed buffer | Returns original object (not buffer) |

---

## 2. Unit specs — `get` (tag-based)

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U6 | Tag with 2 entries | `tags: ["products"]`, 2 keys in tag set | Returns array of 2 decompressed values |
| U7 | Tag with no entries | `tags: ["empty-tag"]` | Returns `[]` |
| U8 | Multiple tags | `tags: ["a", "b"]` | Returns combined entries from all matching tags |
| U9 | Redis connection error | Redis unavailable | Returns `[]`; logs warning; does not throw |

---

## 3. Unit specs — `set`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U10 | Basic set | `{ key, data: { id: 1 }, ttl: 60 }` | Data compressed + stored at `prefix+key`; TTL = 60 |
| U11 | Default TTL used | `ttl` absent | TTL = `options.ttl ?? 3600` |
| U12 | Tags stored | `{ tags: ["products"] }` | `<key>:tags` set; tag set `tag:<hash>` has `keyName` as member |
| U13 | Tag IDs interned | Same tag on 2 sets | Tag ID reused from dictionary (not duplicated) |
| U14 | Data serialized + compressed | Object input | Redis stores compressed binary |
| U15 | Pipeline used | Any set with tags | All commands executed in single pipeline |
| U16 | `options` stored | `{ options: { autoInvalidate: false } }` | `options` field in hash set |
| U17 | Redis connection error | Redis unavailable | Logs warning; no exception thrown |

---

## 4. Unit specs — `invalidate` (by key)

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U18 | Single key invalidation | `"product:1"` | `keyName` and `<key>:tags` deleted from Redis |
| U19 | Tag ref counts decremented | Key with 2 tags | Both tag ref counts decremented by 1 |
| U20 | Key not found | Non-existent key | No error; no-op |

---

## 5. Unit specs — `invalidate` (by tag)

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U21 | Tag with 2 entries | `"products"` tag, 2 keys | Both keys deleted; tag set deleted |
| U22 | `autoInvalidate: false` | Entry opts out | Entry not deleted during tag invalidation |
| U23 | `autoInvalidate: true` (default) | Entry with tag | Entry deleted |
| U24 | Empty tag | Tag exists, no members | Tag set deleted; no errors |

---

## 6. Unit specs — connection loader

| # | Scenario | Expected outcome |
|---|---|---|
| U25 | `redisUrl` missing | Throws `Error("[caching-redis] redisUrl is required")` |
| U26 | Successful PING | Logs `"Redis cache connection test successful"` |
| U27 | Failed PING | Logs warning; does not throw |
| U28 | `redisClient` registered in DI | Container has `redisClient` resolve function |
| U29 | `prefix` registered | Container has `prefix` from options or default `"mc:"` |

---

## 7. Integration specs

| # | Scenario | Expected outcome |
|---|---|---|
| I1 | Set → Get | Value retrievable immediately after set |
| I2 | Set with TTL → Get after expiry | Returns `null` after TTL elapses |
| I3 | Set with tag → Invalidate tag → Get | Returns `null` after tag invalidation |
| I4 | Graceful degradation (Redis down) | `get` returns null; application continues |

---

## 8. Acceptance criteria

- `get` never throws on Redis connection errors; always returns `null` or `[]`.
- `set` never throws on Redis connection errors.
- Tag invalidation deletes all associated keys respecting `autoInvalidate` option.
- All commands use Redis pipelines for atomicity.
- Data is never stored or returned without compression/decompression.
- Default TTL of 3600 seconds is applied when not explicitly specified.
