# Sets

Sets store unique, unordered values. They are handy for tags, memberships, and deduplication.

```redis
SADD course:redis:students alice bob alice
SMEMBERS course:redis:students
SISMEMBER course:redis:students alice
SCARD course:redis:students
```

Adding `alice` twice still stores one member. Remove a member with `SREM`:

```redis
SREM course:redis:students bob
```

Set operations compare groups:

```redis
SADD team:blue alice bob
SADD team:red bob chandra
SINTER team:blue team:red
SUNION team:blue team:red
SDIFF team:blue team:red
```

These return, respectively, members in both teams, either team, and blue but not red.

`SMEMBERS` is great for small sets. For large sets, iterate with `SSCAN` to avoid returning every member in one response.
