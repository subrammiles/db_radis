# Learn Redis

Small, hands-on Redis examples, ordered from beginner to more advanced topics. Run the commands in `redis-cli` after installing and starting Redis (see [Ubuntu installation](install/ubuntu.md)). Lines beginning with `#` are comments, not commands.

## Learning path

1. [Install Redis on Ubuntu](install/ubuntu.md)
2. [Set and get values](b-basic-operations/a-set-and-get.md)
3. [Strings and counters](b-basic-operations/a-set-and-get.md#more-string-examples)
4. [Expiration and TTL](b-basic-operations/b-Set%20Expiration%20(TTL).md)
5. [Lists](b-basic-operations/list.md)
6. [Sets](b-basic-operations/set.md)
7. [Hashes](b-basic-operations/hash.md)
8. [Sorted sets, transactions, and Streams](b-basic-operations/advanced.md)

Redis commands act on keys. Use `TYPE key` to inspect a key's data type and `DEL key` to remove it. These examples use disposable learning keys; avoid running cleanup commands against data you need to keep.
