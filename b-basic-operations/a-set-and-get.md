# Strings: Set and Get

Strings are Redis's basic value type. They can hold text, numbers, or binary data.

Store a value:

```redis
SET user_name "John"
```

Read it back:

```redis
GET user_name
```

Output:

"John"

Delete the key when you no longer need it:

```redis
DEL user_name
```

## More string examples

### Set only when the key does not exist

`NX` is useful for a simple one-time claim:

```redis
SET onboarding:alice done NX
GET onboarding:alice
```

The first `SET` returns `OK`; repeating it while the key exists returns `(nil)`.

### Update a counter

Redis increments an integer atomically:

```redis
SET page:home:views 0
INCR page:home:views
INCRBY page:home:views 4
GET page:home:views
```

### Read and write several keys

```redis
MSET user:1:name "Asha" user:1:city "Pune"
MGET user:1:name user:1:city
```

### Set a value with a lifetime

Set the value and its expiration together (in seconds):

```redis
SET login:token:42 abc123 EX 60
TTL login:token:42
```
