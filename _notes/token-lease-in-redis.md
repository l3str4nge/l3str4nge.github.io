---
title: From Paper to Code Simulating Facebook’s Token Lease in Redis
parent: caching
date: 2026-06-16
summary: Token lease in cache aside pattern
---
### Introduction

Recently I started to learn and explore system design. I was just wondering how people in big tech, way smarter than me, solve different and specific problems.

I was reading about different caching patterns and stumbled upon a great paper from Meta about the cache aside pattern. Here is the original [link](https://research.facebook.com/publications/scaling-memcache-at-facebook/) — I really enjoyed reading through it.

Meta's engineers originally used memcached as their key-value store, at a very, very big scale.

Cache aside is great for heavy read workloads, but it comes with two big problems:

1. Cache stampede
   If a really hot key gets invalidated, the database suddenly takes the full brunt of the traffic that was hitting the cache — the load spikes hard enough that it can take the service down.
2. Race condition
   A stale DB value can end up cached if process A reads from the database on a cache MISS, but process B invalidates the cache in the middle of that operation.

Meta forked memcached and implemented a token lease state machine that solves both problems at once. Since I only really understand something once I've built it (semi vibe-coded, if I'm honest), I couldn't write this article without building a small application first. I'm not good at writing C, though, so I decided to reimplement the idea using Redis instead. Why not?

### How does it work?

In a traditional cache aside pattern, the application talks to both the cache and the database — it acts as the orchestrator. There are a few scenarios to handle:

1. Cache HIT
   Something is already in cache so app returns cached data in the response
2. Cache MISS
   Cache was invalidated so app gets data from database, populates the cache and returns the data.
3. Cache invalidate
   App saves/update new value in the database and invalidates the cache.

Pretty simple right? But what if we have a really hot key cached with 100k requests per second 24/7 and once a week we have to update the value. If we simply delete the key, we move all that huge traffic from the cache straight to the database. We'll kill the DB immediately.

What if someone saves a new value in the database and then reads it, but it doesn't seem to be updated — just because someone else already repopulated the cache with the old value?

So Meta invented a great solution: token lease.

When the application MISSes the cache, it's issued a token. With this token, the app goes to the database, gets the value, then comes back to the cache and populates the key. Only then is the fresh value returned to the caller.

Any other requests that arrive during that window also technically miss, but since a token has already been issued, the cache layer returns a `WAIT` status instead — those requests just poll until the key's status becomes `READY`. If the token owner is lost mid-flight (its lease expires), the next request in line becomes the new owner and the whole cycle repeats — costing one extra DB hit per lost owner. You can imagine a pathological case where every owner in a row gets lost, but that's really a DB availability problem, separate from cache stampede or race conditions, and needs its own solution.

This solves cache stampede by preventing DB load spikes. Here is the Redis function for the `get` operation.

```
local function lease_get(keys, args)
    local key = keys[1]
    local lease_ttl_ms = tonumber(args[1])

    if redis.call('EXISTS', key) == 0 then
        local token = redis.call('HINCRBY', key, 'token', 1)
        redis.call('HSET', key, 'status', 'LEASED')
        redis.call('PEXPIRE', key, lease_ttl_ms)
        return {'MISS', tostring(token), false}
    end

    local status = redis.call('HGET', key, 'status')

    if status == 'READY' then
        local value = redis.call('HGET', key, 'value')
        return {'HIT', false, value}
    end

    if status == 'LEASED' then
        return {'WAIT', false, false}
    end

    -- status == 'STALE'
    local value = redis.call('HGET', key, 'value')
    local issued = redis.call('HGET', key, 'issued')
    if issued ~= '1' then
        redis.call('HSET', key, 'issued', '1')
        local token = redis.call('HGET', key, 'token')
        return {'MISS_STALE_OWNER', token, value}
    end
    return {'STALE_SERVE', false, value}
end
```

Here's the same logic as a state machine:

![Token lease state machine](/assets/notes/token-lease-in-redis/state_machine.png)

You'll notice something called `STALE` in this snippet — wait, don't we want to avoid stale values? Actually, this is a deliberate trade-off. In a normal cache aside pattern, invalidating the cache deletes the key outright. Here, we mark the key's status as `STALE` instead. Here's the race condition scenario this enables us to handle:

1. Cache is invalidated -> status `STALE`
2. Request A gets `MISS_STALE_OWNER` -> gets token, db request is made
3. At the same time request B saves new value in the database and invalidates the cache again -> status `STALE`, already issued token is invalid from this time.
4. Request A gets back to the cache layer and tries to populate the cache with the `old` value but it gets rejected, so the `old` value is returned to the UI.
5. While A's token is leased, every other request in this window gets `STALE_SERVE` from the cache layer, so no DB calls are made and the stale cached value is returned instead. Requests after that will get a proper `HIT` from the cache.
6. The next request repeats the same flow, but this time without the race condition, so the fresh value gets cached and returned properly.

![Race condition sequence](/assets/notes/token-lease-in-redis/race_condition.png)

### Summary

This is how I learned the cache aside pattern, its bottlenecks, and how to approach them. If you want to see the entire example with simulations, I've created a dedicated repository with a small bookstore application: [cache-aside-with-token-lease](https://github.com/l3str4nge/cache-aside-with-token-lease).
