# Expiration (TTL)

TTL means “time to live.” Expiration is useful for temporary sessions, verification codes, and cache entries.

Set a key, then add a 10-second expiration:

SET session "abc123"
EXPIRE session 10

Check the remaining lifetime in seconds:

TTL session

`TTL` returns the remaining seconds, `-1` when the key has no expiration, and `-2` when the key does not exist.

Set a value and expiration in one command to avoid leaving a permanent key if a separate `EXPIRE` command is missed:

```redis
SET reset-code:alice 593104 EX 300
TTL reset-code:alice
```

Other useful commands:

```redis
EXPIRE session 60       # expire after 60 seconds
PEXPIRE session 1500    # expire after 1500 milliseconds
PERSIST session         # remove the expiration
```
