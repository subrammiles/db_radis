# Advanced Redis Notes

These examples build on the basic Redis data types. Use a recent Redis server for the Stream commands. Run them against a local learning instance; production settings depend on the durability and availability your application needs.

## Sorted sets: scores and leaderboards

A sorted set stores unique members ordered by score:

```redis
ZADD game:weekly 120 alice 95 bob 150 chandra
ZREVRANGE game:weekly 0 2 WITHSCORES
ZINCRBY game:weekly 10 bob
ZRANK game:weekly bob
```

`ZREVRANGE` lists highest scores first. `ZRANK` counts upward from the lowest score (starting at zero); use `ZREVRANK` for rank from the top.

## Transactions

`MULTI` queues commands and `EXEC` runs the queued commands together without other clients' commands interleaving:

```redis
MULTI
INCR account:1:events
INCR account:1:events
EXEC
```

For a conditional update, `WATCH` lets you detect whether a key changed before `EXEC`:

```redis
WATCH inventory:item-7
GET inventory:item-7
MULTI
DECR inventory:item-7
EXEC
```

If the watched key changes before `EXEC`, the transaction is aborted. Applications should then re-read and retry when appropriate. `MULTI` does not provide SQL-style rollback for runtime command errors.

## Streams: append-only event records

Streams are useful when events need IDs and consumer groups:

```redis
XADD orders * order_id 501 status created
XADD orders * order_id 502 status created
XRANGE orders - +
```

Create a consumer group from the beginning of the stream, then read new entries as a consumer:

```redis
XGROUP CREATE orders order-workers 0 MKSTREAM
XREADGROUP GROUP order-workers worker-1 COUNT 10 STREAMS orders >
```

After processing entries, acknowledge them so they leave the group's pending list:

```redis
XACK orders order-workers <entry-id>
```

Replace `<entry-id>` with the ID returned by `XREADGROUP`. Production consumers should also handle pending messages and retries.

### Recover stuck entries and cap stream size

If a consumer crashes after reading but before `XACK`, its entries stay pending. Inspect them, then let another consumer claim entries idle for more than 60 seconds (Redis 6.2+):

```redis
XPENDING orders order-workers
XAUTOCLAIM orders order-workers worker-2 60000 0-0 COUNT 10
```

`XAUTOCLAIM` returns a cursor to pass on the next call; `0-0` means the scan is complete. Streams grow until trimmed, so cap them to keep memory bounded:

```redis
XTRIM orders MAXLEN ~ 10000
```

The `~` lets Redis trim approximately, which is more efficient than an exact limit. You can also trim on insert with `XADD orders MAXLEN ~ 10000 * ...`.

## A few useful habits

- Use `SCAN` to explore keys incrementally; `KEYS *` can block a busy server.
- Pick key names that show relationships, such as `user:100:profile`.
- Think about expiration and memory use when storing cache or session data.
- Try examples against a local Redis instance and inspect results with `TYPE`, `TTL`, and the matching data-type commands.

## Pipelining: reduce network round trips

Pipelining sends several commands before waiting for their replies. It reduces network round trips, which can make bulk work much faster. A pipeline is **not** a transaction: other clients' commands may run between its commands.

For a small terminal demonstration, send multiple RESP commands in one input stream:

```sh
printf '*2\r\n$4\r\nINCR\r\n$13\r\ndemo:pipeline\r\n*2\r\n$4\r\nINCR\r\n$13\r\ndemo:pipeline\r\n*2\r\n$3\r\nGET\r\n$13\r\ndemo:pipeline\r\n' | redis-cli --pipe
```

In application code, use the pipeline API provided by your Redis client and read the replies after sending the batch. Keep batches bounded so a large request does not consume too much client or server memory. If later commands need the result of an earlier command, a pipeline cannot remove that dependency; use a short script or separate request instead.

## Lua: make read-and-update logic atomic

Redis runs a Lua script atomically relative to other commands. This is useful when a decision depends on a value that must not change between reading and writing it. Scripts block command processing while they run, so keep them short and avoid loops over unbounded data.

This script deletes a lock only when its stored token matches the caller's token:

```redis
SET lock:invoice:7 random-owner-token NX PX 30000
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:invoice:7 random-owner-token
```

The `SET` reply is `OK` if the lock was acquired, or `(nil)` if another owner has it. Use a fresh, unpredictable token for every acquisition. The script prevents one client from deleting another client's lock after its own lease expires. For high-stakes coordination, account for lease expiry, process pauses, failover, and fencing at the protected resource; a Redis lock alone cannot make an external operation safe.

For frequently used scripts, clients can load one with `SCRIPT LOAD` and call its SHA with `EVALSHA`. Redis Functions (available in Redis 7+) provide a managed server-side alternative. Keep script keys explicit in `KEYS`; this also matters when using Redis Cluster.

## Persistence: decide what can be lost

Redis can persist data with RDB snapshots, AOF command logging, or both. RDB makes point-in-time snapshots; AOF records writes and can usually recover a more recent state, with a performance and disk-space trade-off. The right configuration depends on how much data loss after a crash is acceptable and how quickly the service must recover.

Inspect current persistence status with:

```redis
INFO persistence
LASTSAVE
```

For a manual background RDB snapshot, use `BGSAVE` and then check `LASTSAVE` or `INFO persistence` for completion. A successful snapshot is not a complete backup plan: keep backups outside the Redis host and practice restoring them. Replicas improve availability, but replication is asynchronous and is not a substitute for backups.

## Replication and failover

A replica follows a primary and can serve some read workloads, but replica reads may be stale. Redis replication is asynchronous, so a primary failure can lose writes that had not reached a replica. Sentinel can monitor non-clustered Redis primaries and coordinate failover; Redis Cluster shards data across nodes and also uses replicas for failover.

Check role and replication health with:

```redis
INFO replication
ROLE
```

Choose Sentinel or Cluster based on the availability and sharding needs of the application, then test failover behavior. Do not assume that a successful write reply means the write is already safely stored on disk or copied to every replica.

## Pub/Sub versus Streams

Pub/Sub is live broadcast: subscribers must be connected when a message is published. Messages are not stored for replay and there is no acknowledgment tracking. It fits signals such as cache invalidation or transient presence updates.

Streams store entries and support replay and consumer groups. A consumer group tracks pending entries until they are acknowledged. Choose Streams when consumers may need to recover after disconnects or process work later; plan retention and pending-message recovery as well.

## Redis Cluster: design keys for multi-key work

Cluster divides keys into hash slots. A multi-key command, transaction, or script must use keys in the same slot. Hash tags force related keys into one slot by hashing the text inside `{}`:

```redis
MSET cart:{user-42}:item-1 book cart:{user-42}:item-2 pen
MGET cart:{user-42}:item-1 cart:{user-42}:item-2
```

Both keys share the `{user-42}` tag, so the multi-key command can run in one slot. Hash tags can also concentrate traffic if a tag covers too much data, so choose a tag that matches the multi-key operations you actually need.

## Inspect performance and memory

Use these commands to investigate a local or appropriately authorized server:

```redis
INFO memory
MEMORY USAGE user:100
SLOWLOG GET 10
LATENCY LATEST
```

`SLOWLOG` records commands that exceed the configured execution threshold. Large key scans, unbounded collection reads, expensive scripts, and huge pipeline batches can affect latency. In production, use `SCAN`-family commands for incremental keyspace traversal and monitor memory, latency, evictions, and persistence health over time.

## Sliding-window rate limiter with a sorted set

Store one member per request, scored by its timestamp in milliseconds. Remove entries older than the window, add the current request, and count what remains. Your application computes the timestamps; here "now" is `1764000060000` and the window is 60 seconds:

```redis
MULTI
ZREMRANGEBYSCORE rate:user-42 0 1764000000000
ZADD rate:user-42 1764000060000 req-1764000060000-a1
ZCARD rate:user-42
PEXPIRE rate:user-42 60000
EXEC
```

If the `ZCARD` reply exceeds the limit (for example, 100 requests per minute), reject the request. Members must be unique, so add a random suffix in case two requests share a millisecond. This version also counts rejected requests; move the logic into a Lua script if you only want to record allowed ones.

## HyperLogLog: approximate unique counts

HyperLogLog counts distinct items using at most about 12 KB per key, with a standard error of about 0.81%. It is useful for unique visitors where an exact list is not needed:

```redis
PFADD visitors:2026-11-24 alice bob chandra
PFADD visitors:2026-11-25 bob dev
PFCOUNT visitors:2026-11-25
PFCOUNT visitors:2026-11-24 visitors:2026-11-25
PFMERGE visitors:week visitors:2026-11-24 visitors:2026-11-25
```

Counting several keys together returns the approximate union (here 4). You cannot list the members back out of a HyperLogLog.

## Bitmaps: compact yes/no flags

A bitmap is a string treated as an array of bits. With numeric user IDs, one bit per user tracks daily activity very compactly:

```redis
SETBIT active:2026-11-25 100 1
SETBIT active:2026-11-25 205 1
GETBIT active:2026-11-25 100
BITCOUNT active:2026-11-25
```

Combine days with `BITOP`, for example users active on both days:

```redis
SETBIT active:2026-11-24 100 1
BITOP AND active:both active:2026-11-24 active:2026-11-25
BITCOUNT active:both
```

Bitmaps work best with dense, small integer IDs. Setting a very high bit offset allocates memory up to that offset.

## Geospatial: find nearby places

Geo commands store longitude and latitude (in that order) in a sorted set and search by distance (`GEOSEARCH` needs Redis 6.2+):

```redis
GEOADD stores 80.2707 13.0827 chennai 77.5946 12.9716 bengaluru 73.8567 18.5204 pune
GEODIST stores chennai bengaluru km
GEOSEARCH stores FROMLONLAT 80.2707 13.0827 BYRADIUS 400 km ASC WITHDIST
```

The search returns Chennai and Bengaluru (about 290 km away), nearest first. Pune is outside the radius.

## Expire individual hash fields

Redis 7.4 added per-field expiration for hashes. Older versions can only expire the whole key:

```redis
HSET session:9 user alice csrf t0k3n
HEXPIRE session:9 300 FIELDS 1 csrf
HTTL session:9 FIELDS 2 csrf user
HPERSIST session:9 FIELDS 1 csrf
```

`HTTL` returns `-1` for a field with no expiration and `-2` for a missing field, matching `TTL`. The `FIELDS` count must match the number of field names that follow.

## Further reading

- [Redis pipelining](https://redis.io/docs/latest/develop/using-commands/pipelining/)
- [Redis Lua scripting](https://redis.io/docs/latest/develop/programmability/eval-intro/)
- [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
- [Redis Cluster scaling](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)
- [Distributed locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Redis data types (HyperLogLog, bitmaps, geospatial)](https://redis.io/docs/latest/develop/data-types/)
