# redis-commands

Redis provides a lot of functionality for data types "out of the box." Additional functions can be written via Lua scripts for cases where there isn't 
a "vanilla" function that fits. Redis also uses the Lua interpreter to execute it's native commands. Scripts are executed atomically (blocking).

For more information on executing Lua scripts with Redis see the [Eval Command Documentation](http://redis.io/commands/eval).

Performance is an important consideration when designing software around a data store. Redis has been built as a high performance data store. When
introducing any IO calls into software it's very important to understand the cost of that functionality. Redis documentation includes a *Time complexity* 
factor for each native command so an evaluation can be performed when deciding which commands and data type should be used to solve a problem. Introducing 
custom functionality or commands should be done so with an equal understanding of their cost. Redis Lua scripts are atomically executed so other commands
cannot execute until the script has finished executing. Redis provides a benchmark tool *redis-benchmark* that has been used to test all the commands in 
this package.

For more information on Redis benchmarks see the [Benchmarks Documentation](http://redis.io/topics/benchmarks).

## Usage

Redis Lua scripts are located in the `./src` directory.

Use them directly or use a language/environment implementation.

### Language/Environment Documentation

- [NodeJS](docs/nodejs.md)

## Commands

Redis Lua scripts provided by this package.

### Priority Lists

Custom Redis data type that is comprised of a keyspace of lists that are ordered by priority. Priority lists (plist) are native Redis Lists with all the 
standard [List Commands](http://redis.io/commands#list).

#### PLPUSH key plist value [value ...]

Insert all the specified values at the head of the plist stored at key:plist. If key:plist does not exist, it is created as an empty list before performing 
the push operations. When key:plist holds a value that is not a list, an error is returned.

It is possible to push multiple elements using a single command call just specifying multiple arguments at the end of the command. Elements are inserted 
one after the other to the head of the list, from the leftmost element to the rightmost element. So for instance the command PLPUSH mylist high a b c will result 
into a list containing c as first element, b as second element and a as third element.

**Time complexity**

O(1)

**Return values**

[Integer reply](http://redis.io/topics/protocol#integer-reply): the length of the list after the push operations.

**Examples**

```
> redis-cli script load "$(cat /path/to/redis-commands/src/plpush.lua)"
"da39a3ee5e6b4b0d3255bfef95601890afd80709"
redis> EVALSHA da39a3ee5e6b4b0d3255bfef95601890afd80709 2 mylist high "hello"
(integer) 1
redis> EVALSHA da39a3ee5e6b4b0d3255bfef95601890afd80709 2 mylist high "world"
(integer) 1
redis> LRANGE mylist:high 0 -1
1) "hello"
2) "world"
```

#### PLLEN key plist [plist ...]

Returns the lengths of the plist(s) for each key:plist. If key:plist does not exist, it is interpreted as an empty plist and 0 is returned. An error is returned when the value stored at key:plist is not a list.

**Time complexity**

O(1 * N) where N is the number of priority lists to try by the operation.

**Return values**

[Array reply](http://redis.io/topics/protocol#array-reply): list of plist(s) in the specified order with list length.

**Examples**

```
> redis-cli script load "$(cat /path/to/redis-commands/src/pllen.lua)"
"09ca92a0ded4a33398413bb4a22a3f1ef45c0c89"
redis> LPUSH mylist:critical "hello"
(integer) 1
redis> LPUSH mylist:low "world"
(integer) 1
redis> EVALSHA 09ca92a0ded4a33398413bb4a22a3f1ef45c0c89 5 mylist critical high medium low
1) "critical"
2) (integer) 1
3) "high"
4) (integer) 0
5) "medium"
6) (integer) 0
7) "low"
8) (integer) 1
```

#### PLREMLPUSH source destination plist count value

Removes the first count occurrences of elements equal to value from the plist stored at source:plist and pushes the elements at the head of the plist stored 
at destination:plist.

The count argument influences the operation in the following ways:

- count > 0: Remove elements equal to value moving from head to tail.
- count < 0: Remove elements equal to value moving from tail to head.
- count = 0: Remove all elements equal to value.

For example, PLREMLPUSH source destination medium -2 "hello" will remove the last two occurrences of "hello" in the plist stored at source:plist and push them at the head of plist at 
destination:plist.

Note that non-existing keys are treated like empty plists, so when key does not exist, the command will always return 0.

**Time complexity**

O(N) where N is the length of the list.

**Return values**

[Integer reply](http://redis.io/topics/protocol#integer-reply): the number of removed elements.

**Examples**

```
> redis-cli script load "$(cat /path/to/redis-commands/src/plremlpush.lua)"
"58076efaa93d06bd0d68688c9bab745696d5cb18"
redis> LPUSH source:critical "string"
(integer) 1
redis> LPUSH source:medium "string"
(integer) 1
redis> LPUSH source:critical "string"
(integer) 2
redis> EVALSHA 58076efaa93d06bd0d68688c9bab745696d5cb18 3 source destination critical 0 "string"
(integer) 2
redis> LLEN source:critical
(integer) 0
redis> LLEN destination:critical
(integer) 2
```

#### PRPOPLPUSH source destination plist [plist ...]

Atomically returns and removes the last element (tail) of the priority list (plist) stored at source:plist, and pushes the element at the first element (head) of the plist stored at 
destination:plist.

For example: consider source holding the plist critical a,b,c, and destination holding the list critical x,y,z. Executing PRPOPLPUSH results in source:critical holding a,b and 
destination:critical holding c,x,y,z.

If source does not exist, the value nil is returned and no operation is performed. If source and destination are the same, the operation is equivalent to removing the last element from the 
list and pushing it as first element of the list, so it can be considered as a list rotation command.

Most use cases will provide multiple plists in which case executing PRPOPLPUSH will attempt the operation on each plist in order until success.

**Time complexity**

O(1 * N) where N is the number of priority lists to try by the operation.

**Return values**

[Bulk string reply](http://redis.io/topics/protocol#bulk-string-reply): the element being popped and pushed.

**Examples**

```
> redis-cli script load "$(cat /path/to/redis-commands/src/prpoplpush.lua)"
"958b2c29c81d76dc6d5978e5255a16a9782e6a76"
> redis-cli
redis> LPUSH source:high "one"
(integer) 1
redis> LPUSH source:medium "two"
(integer) 1
redis> LLEN source:high
(integer) 1
redis> LLEN source:medium
(integer) 1
redis> LLEN destination:high
(integer) 0
redis> LLEN destination:medium
(integer) 0
redis> EVALSHA 958b2c29c81d76dc6d5978e5255a16a9782e6a76 6 source destination critical high medium low
"one"
redis> EVALSHA 958b2c29c81d76dc6d5978e5255a16a9782e6a76 6 source destination critical high medium low
"two"
redis> LLEN source:high
(integer) 0
redis> LLEN source:medium
(integer) 0
redis> LLEN destination:high
(integer) 1
redis> LLEN destination:medium
(integer) 1
```

### Hashes

Native Redis data type. [Hash Commands](http://redis.io/commands#hash).

#### HGETRAND key

Returns a random value in the hash stored at key.

**Time complexity**

O(N) where N is the size of the hash.

**Return value**

[Bulk string reply](http://redis.io/topics/protocol#bulk-string-reply): the random value, or nil when key does not exist.

**Examples**

```
> redis-cli script load "$(cat /path/to/redis-commands/src/hgetrand.lua)"
"0565bb477f320b323aaea486befebd3f9a079176"
> redis-cli
redis> HMSET myhash one "alpha" two "beta" three "charlie"
OK
redis> EVALSHA 0565bb477f320b323aaea486befebd3f9a079176 1 myhash
"beta"
```

#### HMGETRAND key count

Returns a list of random values in the hash stored at key.

List size equals count or hash length when count is greater than hash length.

**Return value**

[Array reply](http://redis.io/topics/protocol#array-reply): list of random values.

**Examples**

```
> redis-cli script load "$(cat /path/to/redis-commands/src/hmgetrand.lua)"
"e8ac5ad65be171300c60f830dc6c974d524129bc"
> redis-cli
redis> HMSET myhash one "alpha" two "beta" three "charlie"
OK
redis> EVALSHA e8ac5ad65be171300c60f830dc6c974d524129bc 1 myhash 3
1) "charlie"
2) "beta"
3) "alpha"
```

### Sorted Sets

Native Redis data type. [Sorted Set Commands](http://redis.io/commands#sorted_set).

#### ZRANGEINCRBY key start stop increment [WITHSCORES]

Returns the specified range of elements in the sorted set stored at key after incrementing the score. The elements are considered to be ordered from the lowest to the highest score. Lexicographical order is used for elements with equal score.

Both start and stop are zero-based indexes, where 0 is the first element, 1 is the next element and so on. They can also be negative numbers indicating offsets from the end of the sorted set, with -1 being the last element of the sorted set, -2 the penultimate element and so on.

start and stop are inclusive ranges, so for example `ZRANGEINCRBY myzset 0 1 4` will return both the first and the second element of the sorted set after incrementing their scores by four.

Out of range indexes will not produce an error. If start is larger than the largest index in the sorted set, or start > stop, an empty list is returned. If stop is larger than the end of the sorted set Redis will treat it like it is the last element of the sorted set.

It is possible to pass the WITHSCORES option in order to return the new scores of the elements together with the elements. The returned list will contain value1,score1,...,valueN,scoreN instead of value1,...,valueN. Client libraries are free to return a more appropriate data type (suggestion: an array with (value, score) arrays/tuples).

**Time complexity**

O(log(N)+M) with N being the number of elements in the sorted set and M the number of elements returned.

**Return value**

[Array reply](http://redis.io/topics/protocol#array-reply): list of elements in the specified range (optionally with their scores, in case the WITHSCORES option is given).

**Examples**

```
> redis-cli script load "$(cat /path/to/redis-commands/src/zrangeincrby.lua)"
"009e91e9caab43c31bcc48843ca05d8bb951eb7d"
> redis-cli
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZADD myzset 2 "two"
(integer) 1
redis> ZRANGE myzset 0 1 WITHSCORES
1) "one"
2) "1"
3) "two"
4) "2"
redis> EVALSHA 009e91e9caab43c31bcc48843ca05d8bb951eb7d 1 myzset 0 0 1 WITHSCORES
1) "one"
2) "2"
redis> ZRANGE myzset 0 1 WITHSCORES
1) "one"
2) "2"
3) "two"
4) "2"
```

#### ZLPOPRPUSH source destination

Atomically returns and removes the first element (head) of the sorted set stored at source, and pushes the element at the last element (tail) of the sorted set stored 
at destination. Sorted Set rules still apply.

For example: consider source holding the sorted set a 1, b 2, c 3, and destination holding the sorted set x 4, y 5, z 6. Executing ZLPOPRPUSH results in source holding b 2, c 3 and destination 
holding a 1, x 4, y 5, z 6.

If source does not exist, the value nil is returned and no operation is performed. If source and destination are the same, the operation is equivalent to 
removing the first element from the sorted set and pushing it as last element of the sorted set, so it can be considered as a sorted set rotation command.

Note: Testing revealed that when compared to the O(1) RPOPLPUSH List function with 10M member sets requests per second was reduced by 66%

**Return value**

[Array reply](http://redis.io/topics/protocol#array-reply): element popped and pushed with score.

**Examples**

```
> redis-cli script load "$(cat /path/to/redis-commands/src/zlpoprpush.lua)"
"a34f28bab1fdcd6ca9effe3ce21f797c4873024f"
> redis-cli
redis> ZADD source 1 "one"
(integer) 1
redis> ZADD source 2 "two"
(integer) 1
redis> ZCARD source
(integer) 2
redis> ZCARD destination
(integer) 0
redis> EVALSHA a34f28bab1fdcd6ca9effe3ce21f797c4873024f 2 source destination
1) "one"
2) "1"
redis> EVALSHA a34f28bab1fdcd6ca9effe3ce21f797c4873024f 2 source destination
1) "two"
2) "2"
redis> ZCARD source
(integer) 0
redis> ZCARD destination
(integer) 2
```

## Command Requests

Cannot find the command you need?

[File an Enhancement/Issue](https://github.com/gregl83/redis-commands/issues).

## License

MIT