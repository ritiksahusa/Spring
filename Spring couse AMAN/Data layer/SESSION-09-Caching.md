# Session 9 — Caching

**Track:** JPA / Hibernate SDE-2 Mastery  
**Scope:** first-level cache -> second-level cache -> query cache -> invalidation -> consistency -> cache choice

## Session Outcome

You should be able to:

- distinguish first-level, second-level, query, and application caches;
- explain cache scope and identity behavior;
- describe what is cached and what is not;
- reason about invalidation after updates, bulk DML, and transactions;
- choose cache concurrency strategy from data mutability and consistency needs;
- identify stale-data, stampede, memory, and invalidation risks;
- explain why caching a query result is different from caching entities;
- decide when caching helps and when it hides a database/design problem.

---

## 1. The Cache Layers

```text
request/service
    -> first-level cache: one persistence context / EntityManager
    -> second-level cache: shared by sessions in one SessionFactory/EMF
    -> query cache: query result keys/identifier lists, if enabled
    -> database
    -> application/distributed cache: separate design and consistency boundary
```

Each layer has a different scope, lifecycle, and invalidation model. Saying “Hibernate cache” without naming the layer is an incomplete answer.

| Cache | Scope | Typical content | Default assumption |
| --- | --- | --- | --- |
| First-level | one persistence context | managed entity identity/state | normally present |
| Second-level | SessionFactory/EntityManagerFactory | entity/collection/natural-id data | provider/configuration dependent |
| Query cache | shared provider cache | query key -> IDs/scalar result metadata | usually separately enabled |
| Application cache | service/process/distributed | arbitrary response/domain data | application-owned |

---

## 2. First-Level Cache Recap

Within one persistence context:

```java
Order first = entityManager.find(Order.class, id);
Order second = entityManager.find(Order.class, id);
assert first == second;
```

The first-level cache provides identity and avoids repeated database loads for the same entity key in that context. It is also the tracking context for dirty checking.

It is not a shared cache:

- another request has another context;
- another transaction can see different data according to isolation;
- `clear()` affects only the current context;
- closing the context removes the local identity map.

Do not use a first-level cache hit as proof that the database or another node has current data.

---

## 3. Second-Level Cache Scope

A second-level cache is shared across persistence contexts associated with a SessionFactory/EntityManagerFactory. It can avoid database reads for eligible entity/collection data after one context has loaded it.

```text
Session A loads Product 7 -> database -> L2 put
Session B loads Product 7 -> L2 hit, no database read
```

The entity returned to Session B is still managed by Session B; the cache entry is not the managed object itself. Hibernate uses cache data to hydrate a session-specific entity.

### What can be cached

Depending on provider/configuration:

- entity state by identifier;
- collection association keys;
- natural-id resolution;
- query result keys/regions;
- timestamps/update metadata.

Associations and entity contents can have separate cache regions and policies. Enabling an entity region does not mean every graph is cached as one object.

### Shared does not mean universally distributed

In a single JVM, the cache may be local. In a cluster, each node needs an appropriate provider/invalidation or replicated strategy. A local cache can return different data on different nodes after a write unless invalidation/versioning is coordinated.

---

## 4. Cache Consistency and Invalidation

A cache must answer:

- when is an entry populated?
- when is it invalidated or updated?
- what happens on rollback?
- how are concurrent writers coordinated?
- what is the allowed staleness?
- what happens after bulk SQL or an out-of-band database writer?

Hibernate normally coordinates cache changes with transaction outcomes according to its strategy/provider. A transaction that rolls back should not leave a committed-looking cache entry for its uncommitted state, but exact guarantees depend on configuration.

### Out-of-band writes

If a reporting job, migration, native SQL script, or another service updates a table without participating in the cache's invalidation protocol, cached data can remain stale. Options include:

- evict affected regions;
- disable caching for that data;
- route all writes through a coordinated path;
- use a cache with an explicit external invalidation mechanism;
- accept bounded staleness and document it.

A cache is part of the write architecture, not merely a read optimization.

---

## 5. Entity Cache versus Query Cache

### Entity cache

Maps an entity type and identifier to cached state:

```text
(Product, 7) -> product state
```

A query can still hit the database to find which IDs match, then use entity-cache entries for hydration.

### Query cache

Maps query parameters and metadata to a result representation, commonly a list of entity identifiers or scalar results:

```text
(query shape + parameters + pagination) -> [7, 9, 12]
```

It often depends on update-timestamp information to know whether relevant tables changed. A query-cache hit may still require entity cache hits; otherwise entities are loaded separately.

### Why query cache is fragile

- high query-parameter cardinality creates many entries;
- frequently changing tables invalidate results often;
- pagination creates separate keys;
- bulk/native writes can make invalidation hard;
- memory and eviction behavior can surprise;
- query cache overhead can exceed saved work.

Do not enable it as a blanket fix for slow queries. Measure hit ratio, invalidation rate, database work, and latency.

---

## 6. Cache Concurrency Strategies

Provider terminology varies, but the conceptual choices are:

| Strategy | Suitable data | Trade-off |
| --- | --- | --- |
| Read-only | immutable/reference data | fast, writes unsupported/unsafe |
| Non-strict read/write | rarely changing data with tolerated staleness | possible stale reads |
| Read/write | mutable data needing coordinated cache updates | more coordination/overhead |
| Transactional | strong transactional cache semantics where supported | provider/resource complexity |

Choose from business consistency:

- currency exchange reference data may be immutable for a release window;
- product descriptions may tolerate bounded staleness;
- inventory availability generally should not rely on a stale entity cache for correctness;
- permissions/entitlements need explicit consistency analysis.

A cache cannot repair a missing version/lock strategy for concurrent writes.

---

## 7. Collections and Association Caches

A cached collection often stores the association's member identifiers, not every child object's full state. Loading a cached order's line IDs can still require entity cache/database work for each line.

Association invalidation can be frequent:

- adding/removing a child invalidates the collection entry;
- orphan removal changes both entity and collection state;
- bulk changes can bypass normal association notifications;
- large collections create expensive cache entries and invalidations.

Avoid caching huge, frequently changing collections merely because reads are common. A query/projection or bounded page may be more appropriate.

---

## 8. Bulk DML and Cache Staleness

Bulk JPQL/native SQL bypasses per-entity state tracking and may not automatically evict every affected second-level/query cache entry in the way a normal entity update does.

After bulk DML, consider:

- flush pending changes first;
- clear the current persistence context;
- evict affected entity/collection/query regions if required;
- reload before continuing;
- document cache invalidation for native/out-of-band writes.

Example risk:

```text
L2 contains Product 7 price=10
bulk SQL changes all prices to 12
next request reads L2 -> price=10 unless invalidated
```

The exact provider behavior should be verified with an integration test using the actual cache configuration.

---

## 9. Hibernate L2 Cache versus Spring `@Cacheable`

These are different abstractions:

- Hibernate second-level cache stores ORM entity/collection/query data around persistence-context loading.
- Spring Cache `@Cacheable` stores method return values according to application cache configuration.

Caching a service DTO with `@Cacheable` does not automatically invalidate a Hibernate entity region, and an entity-cache hit does not populate a service response cache with the same key/shape.

Application cache design must define:

- key and tenant scope;
- TTL/eviction;
- serialization/versioning;
- invalidation on writes;
- stampede protection;
- authorization-sensitive data boundaries.

Do not stack caches without a reason. Multiple layers make stale-data diagnosis harder.

---

## 10. Cache Stampede and Hot Keys

When a popular entry expires, many requests can miss simultaneously and load the same data. This is a cache stampede.

Mitigations include:

- jittered TTLs;
- refresh-ahead;
- single-flight/request coalescing;
- bounded locking around reload;
- stale-while-revalidate where allowed;
- prewarming stable reference data.

A hot key can also create eviction pressure and contention. Measure hit rate and load amplification, not only average latency.

---

## 11. When Caching Helps and Hurts

### Good candidates

- read-heavy, stable reference data;
- repeated lookup by identifier;
- expensive, deterministic reads with manageable invalidation;
- data where bounded staleness is acceptable;
- data with high cache hit ratio and small entries.

### Poor candidates

- rapidly changing inventory or balances used for correctness;
- huge entities/collections;
- high-cardinality query parameters;
- data with frequent bulk/native updates;
- tenant/user-specific data without safe key isolation;
- data already cheap compared with cache coordination overhead.

A slow endpoint caused by N+1, missing indexes, or an inefficient query should usually be fixed at that layer before adding cache complexity.

---

## 12. Scenario Reasoning

### Scenario A: second request is fast but another node is stale

A local L2 cache is used on two application nodes. A write on node A is immediately visible there but node B serves old data.

**Answer:** The cache is not coordinated across nodes. Choose invalidation/replication, route consistency-sensitive reads appropriately, evict on write, or do not use the cache for that data.

### Scenario B: bulk price update

A native SQL job updates product prices, but the application continues returning old prices.

**Answer:** The write bypassed ORM/cache invalidation. Evict affected regions or disable/coordinate caching for that write path, and clear/reload current contexts.

### Scenario C: query cache enabled, no improvement

A search query has many filters and page values.

**Answer:** High key cardinality and frequent invalidation can yield a low hit ratio. Inspect query-cache hits/misses and entity-cache hits; optimize query/index/fetch shape first.

### Scenario D: caching inventory

A team caches available stock to reduce database reads, then decrements stock concurrently.

**Answer:** A stale cache cannot enforce stock correctness. Use database transaction/version/lock/conditional-update semantics for the invariant; cache only a non-authoritative display value if acceptable.

---

## 13. SDE-2 Interview Answers

### First-level versus second-level cache?

First-level cache is the persistence context for one EntityManager/session and provides identity plus tracking. Second-level cache is shared across sessions in a SessionFactory/EntityManagerFactory and can supply entity/collection data, subject to provider and consistency configuration.

### What does the query cache store?

It stores a query key and result representation, commonly identifiers or scalar results, not necessarily full entity state. Entity cache/database work may still be needed, and invalidation can be expensive.

### When should you not cache?

When data changes frequently, correctness depends on current state, entries are huge/high-cardinality, writes bypass invalidation, or the underlying query/index problem has not been fixed. Cache only with a clear staleness and invalidation contract.

### How does bulk DML affect caches?

It bypasses normal per-entity notifications and can leave persistence-context, second-level, and query-cache data stale. Flush/clear and explicit eviction/reload may be required according to actual provider configuration.

---

## 14. Practice Batch

1. Why is a first-level cache hit not shared across requests?
2. Why can query cache still cause database/entity-cache work?
3. What is the consistency risk of caching inventory?
4. How do native writes affect L2/query caches?
5. What metrics would you inspect before keeping a query cache?

### Model answer key

1. It belongs to one persistence context/EntityManager, normally scoped to a unit of work. Another request has a different context.
2. The query cache may return IDs or scalar metadata, after which entities may need L2 or database loading. The result also has invalidation/parameter-key costs.
3. A stale value can lead to incorrect allocation decisions. Correctness must come from transactional version/lock/conditional-update logic, not a cache.
4. They may bypass ORM invalidation and leave old entries in current contexts, L2, or query regions. Evict/coordinate or avoid caching that data.
5. Hit/miss ratio, invalidation frequency, entry size, latency with/without cache, database load, stampede behavior, memory/evictions, and correctness/staleness incidents.

---

## Session Handoff

**Prepared study material:** Session 9 — Caching  
**Topics covered:** first-level cache, second-level cache, query cache, entity/collection regions, invalidation, concurrency strategies, bulk DML, multi-node consistency, application-cache distinction, stampede risk, and cache selection.  
**Next session:** Session 10 — Production Debugging  
**Next starting topic:** symptom -> evidence -> hypothesis -> experiment for N+1, pool exhaustion, long transactions, stale state, and slow SQL

## One-Line Revision Summary

Caching is a consistency contract with performance benefits: name the layer, scope, contents, hit/invalidation behavior, and staleness tolerance before enabling it.
