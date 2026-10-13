# Lists

Lists preserve order and can be used for simple queues or recent-item histories.

```redis
LPUSH tasks "write docs"
LPUSH tasks "review changes"
LRANGE tasks 0 -1
```

`LPUSH` adds to the head (left). `RPOP` removes from the tail (right), so this pair processes the oldest item first:

```redis
RPOP tasks
LLEN tasks
```

For a queue that waits until an item is available, use `BLPOP` with a timeout in seconds (`0` waits indefinitely):

```redis
LPUSH jobs "job-001"
BLPOP jobs 5
```

`LRANGE key 0 -1` displays the full list. In real applications, choose a bounded range so a very large list does not produce a huge response.
