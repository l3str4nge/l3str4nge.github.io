---
title: Understanding deadlocks for the first time in my life
parent: databases
date: 2026-09-07
summary: How deadlocs are detected in PostgreSQL + simple visualisation
---
### Introduction

It's been a while. I've always been poor at databases. I always avoided this topic. But recently I started to see a lot of deadlocks errors in sentry at work. So having no idea what to learn next I was like: okay let's deal with deadlocks.

### How does deadlocks work?

Suppose we have a table `items` with 2 columns: id and value.
Suppose we create 2 independed transactions:

First transaction (T1):

```sql
BEGIN;
UPDATE items SET value = 10 where id = 1;
# NO COMMIT command so far

```

Now in the meantime we have a second transaction (T2):

```sql
BEGIN;
UPDATE items set value = 20 where id = 2;
# NO COMMIT command so far
```

Then T1 continues with updating id=2

```sql
...
UPDATE items set value = 30 where id = 2;
```

Above command hangs because T2 already blocked row (`id=2`) so T1 waits.

Then T2 continues with updating id=1 

```sql
UPDATE items set value = 40 where id = 1;
```

And...

```sql
ERROR:  deadlock detected
DETAIL:  Process 8968 waits for ShareLock on transaction 753; blocked by process 8953.
Process 8953 waits for ShareLock on transaction 754; blocked by process 8968.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,12) in relation "items"
```





This is my pretty visual explanation 😄

![Deadlock detection](/assets/notes/deadlock-detection/deadlock-detection.png)


### How it works in PostgreSQL?

In it's official repository this [readme](https://github.com/postgres/postgres/blob/master/src/backend/storage/lmgr/README) explains this as a directed graph (the waits-for grapgh) [source](https://github.com/postgres/postgres/blob/master/src/backend/storage/lmgr/README#L393). In other words each transactions is a node, when particular transaction waits for an another one we create an edge. Deadlock is detected when the `cycle` is detected. Cycle means that during graph traversal we jump again to the visited node. Example: T1 -> T2 -> T3 -> T2 (T2 was before and we got back to it again -> error).

Interesting part is that PSQL doesn't trigger DFS search everytime new transaction/edge is created. Instead there is a ``deadlock_timeout`` variable that decides when to trigger a cycle detection [source](https://www.postgresql.org/docs/current/runtime-config-locks.html#GUC-DEADLOCK-TIMEOUT) and by default it's a 1 second. This is optimistic approach that assumes that server gives enough time to unlock transcations by the application naturaly.

I did create a visualisation with DFS implementation inside how exactly it works. [Check it out!](https://deadlock-detection.streamlit.app/)

### How to avoid deadlocks
