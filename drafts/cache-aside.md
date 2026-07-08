## DIRTY NOTES ABOUT CACHE ASIDE

This is a technique where an application talks to either cache and database so it's a orchestrator and handles cache logic.

Cache HIT workflow:

1. User makes a request
2. App checks the cache (key exists -> HIT)
3. App returns data from cache to user

Cache MISS workflow

1. User makes a request
2. App checks the cache (key doesn't exist -> MISS)
3. App get data from DB
4. App saves data to cache
5. App returns data to user

Idempodency -> you do something several times but effect is the same just like you'd do it once.

Thundering herd (cache stampede / dog-pile effect) -> very popular key is evicted from cache (or TTL reached 0) and in the same time tremendous number of requests are made to get this key.

Stale set -> caused by race condition:

1. Process A reads cache with cache MISS
2. Process A reads data from DB
3. Process B writes new entry in the same time (saves in persistance storage and delete non existing key from cache)
4. Process A has old data from DB (because process B saved just after) and set new key in cache (with old value)
5. Result -> DB has new version, cache has old version.

Facebook ida how to prevent stale sets:

1. Client A reads the data and get cache miss and recieves a special token
2. In the same time client B writes data and invalidates token for that key
3. Client A reads data from DB (before client B writes) and tries to save it into cache
4. Cache checks the token which was invalidated by client B request -> data is not set in cache
5. Old DB value is returned to the customer (expected).

Idea how to handle a failures. When multiple cache servers are gone then clients hit DB more frequently so the overall DB load increase
and we can get cache stampede effect.
Facebook decided to chose 1% of machines that will behave like gutter and in 99% of the time these machines do nothing, however in case of failure they acts
as temporary cache. Worklow is as follow:

1. Client read cache -> timeout
2. Instead of hitting database they hit gutter
3. If key exists in the gutter that's fine, if not then DB request is needed and key is saved in gutter.

Most important thing in gutter: very low TTL because invalidation systems know nothing about gutters data. This is example of trading of availability over the consistency. It's better to server unfresh data for couple minutes instead of 500 errors page.

In above scenario consistent hashing won't work because of hot spots, preventing from redirecting huge load to another node (it will cause cascading failure).
