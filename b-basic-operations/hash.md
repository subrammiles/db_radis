# Hashes

Hashes store fields and values under one Redis key. They work well for simple records such as a user profile.

```redis
HSET user:100 name "Mina" city "Chennai" level 1
HGET user:100 name
HGETALL user:100
```

Update one field and increment a numeric field:

```redis
HSET user:100 city "Kochi"
HINCRBY user:100 level 1
```

Check for a field and remove it:

```redis
HEXISTS user:100 email
HDEL user:100 city
```

`HGETALL` is convenient for small hashes. For large hashes, use `HSCAN` to iterate in batches. Redis does not automatically expire individual hash fields; expiration applies to the whole key.
