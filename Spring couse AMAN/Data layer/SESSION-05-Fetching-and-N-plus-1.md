# Session 5 — Fetching and the N+1 Query Problem

**Track:** JPA / Hibernate SDE-2 Mastery  
**Scope:** lazy loading -> proxies -> `LazyInitializationException` -> N+1 -> fetch joins -> entity graphs -> batching -> pagination

## Session Outcome

You should be able to:

- explain lazy loading as a runtime fetch decision;
- distinguish an uninitialized proxy/collection from a null association;
- diagnose N+1 using request traces and SQL counts;
- choose among fetch joins, entity graphs, DTO projections, and batch fetching;
- explain why EAGER is not a universal fix;
- recognize pagination and collection-fetch pitfalls;
- design an endpoint-specific fetch plan;
- fix `LazyInitializationException` without blindly widening persistence-context scope.

---

## 1. Fetching Is a Query-Shape Decision

A relationship mapping says what can be navigated. A fetch plan says what this use case needs now.

```text
endpoint requirement
        -> required fields/relationships
        -> query/fetch plan
        -> SQL shape and row count
        -> mapping/result conversion
```

Do not start with “make everything EAGER.” Start with:

- Which fields does the response need?
- How many root rows are expected?
- Which relationships are to-one or collections?
- Can a join multiply rows?
- Is the result paginated?
- Is the result read-only and better represented as a DTO?

A good mapping and a good query can still be different things. One entity can have several valid fetch plans.

---

## 2. Lazy Loading

With a lazy association, Hibernate may place a proxy or persistent collection wrapper in the entity. The associated row/collection is loaded when accessed while the persistence context can service the request.

```java
Order order = orderRepository.findById(id).orElseThrow();
// customer may not have been selected yet
String name = order.getCustomer().getName(); // may trigger SELECT
```

For collections:

```java
Order order = ...;
order.getLines().size(); // may initialize the collection
```

The Java reference existing does not prove that database data has been loaded. `null`, an uninitialized proxy, and an initialized empty collection are different states.

### Lazy loading is not free

It postpones work; it does not remove work. If code accesses a relationship once per root row, lazy loading can create N+1. If it accesses outside the persistence context, it can fail.

### Initialization checks

Hibernate provides provider-specific utilities such as `Hibernate.isInitialized(...)`, but application code should usually rely on a deliberate fetch plan rather than sprinkling initialization checks throughout controllers.

---

## 3. `LazyInitializationException`

A typical failure:

```java
@Transactional(readOnly = true)
public OrderDto getOrder(Long id) {
    Order order = repository.findById(id).orElseThrow();
    return mapper.toDto(order);
}

// mapper is called after the transaction/context is closed in another design:
// order.getLines() -> LazyInitializationException
```

The exception means a lazy association needed a persistence context/session that is no longer available. It is a boundary/fetch-plan problem, not proof that lazy loading itself is bad.

### Correct fixes

- load the needed graph inside the service transaction;
- use a JPQL fetch join when result cardinality is safe;
- use `@EntityGraph` for a named fetch plan;
- use a DTO/projection query that selects exactly the response shape;
- use batch fetching when multiple roots need the same lazy association.

### Weak fixes

- switch every association to EAGER;
- enable OSIV without measuring SQL;
- call `toString()` or getters randomly to force initialization;
- keep the transaction open through remote calls or serialization.

The correct fix is determined by the endpoint's data contract.

---

## 4. N+1: Definition and Detection

N+1 occurs when one query loads N root rows and then an additional query is executed for each root or relationship access.

```text
1 query: select orders ...
N queries: select customer ... where id = ?
Total: N + 1
```

Example:

```java
List<Order> orders = repository.findRecent(pageable); // 1 query
for (Order order : orders) {
    response.add(order.getCustomer().getName());     // N lazy queries
}
```

### Diagnose from evidence

Look for:

- repeated SQL differing only by a bind parameter;
- query count growing linearly with result size;
- a trace showing one repository call followed by many ORM selects;
- high database round-trip time despite small individual queries;
- endpoint latency that grows with page size.

Enable SQL and bind logging carefully in non-production or sampled production paths. Pair it with metrics such as query count per request, database time, and result cardinality. Logs without a request correlation ID are hard to attribute.

### N+1 is not only lazy collections

It can arise from:

- lazy to-one relationships;
- lazy collections;
- entity-to-DTO mapping;
- template/JSON serialization;
- authorization checks in a loop;
- repository calls inside loops;
- nested relationship traversal.

Fix the access pattern, not only one annotation.

---

## 5. Fetch Join

A JPQL fetch join loads the association as part of the query:

```java
@Query("""
       select o
       from Order o
       join fetch o.customer
       where o.status = :status
       order by o.createdAt desc
       """)
List<Order> findWithCustomer(OrderStatus status);
```

This can turn one root query plus N customer queries into one SQL join. It is a strong tool for to-one relationships and carefully chosen collection loads.

### Collection fetch join risks

A collection join multiplies rows:

```text
1 order with 10 lines -> 10 result rows at SQL level
```

The ORM may de-duplicate the root entity, but the database still processes the expanded row set. Multiple collection fetch joins can create a Cartesian explosion or provider limitations such as multiple-bag issues.

Use `distinct` when appropriate for entity results, but understand that it does not make the underlying relational work disappear.

### Fetch join plus pagination

Paging a collection fetch join is dangerous because SQL rows represent joined children, not clean root entities. Page boundaries can be incorrect or expensive. Safer alternatives:

1. page root IDs;
2. fetch the graph for those IDs in a second query;
3. restore order and map the response.

---

## 6. Entity Graphs

An entity graph defines an attribute fetch plan without embedding every join in query text:

```java
@EntityGraph(attributePaths = {"customer", "lines"})
Optional<Order> findDetailedById(Long id);
```

Advantages:

- keeps a repository method readable;
- makes required attributes explicit at the use-case method;
- can be reused as a named graph;
- avoids globally changing mapping defaults.

Limitations:

- provider and query behavior still need verification;
- collection graphs can still multiply rows;
- a graph does not automatically solve pagination or response overfetching;
- nested graph design needs care.

Use entity graphs for a clear entity-shaped fetch plan. Use DTO projections when the endpoint needs a narrow, stable shape or aggregation.

---

## 7. Batch Fetching

Batch fetching groups lazy loads so that accessing relationships for many roots results in fewer `IN (...)` queries:

```text
without batching: select customer where id = 1
                   select customer where id = 2
                   ...
with batching:    select customer where id in (1, 2, ..., k)
```

Hibernate can use configuration or `@BatchSize`:

```java
@ManyToOne(fetch = FetchType.LAZY)
@BatchSize(size = 32)
private Customer customer;
```

Batch fetching is not the same as a fetch join:

- fetch join changes the root query shape;
- batch fetching groups deferred loads.

It can be valuable when the service traverses a known set of roots but a join would create too many rows. Tune the batch size against database parameter limits and actual workload.

### Subselect fetching

Provider-specific subselect strategies can load associations for all roots from a previous query in one follow-up query. They can help some read patterns but may have surprising scope and memory costs. Measure before adopting.

---

## 8. DTO and Projection Queries

For a list endpoint, selecting entities may load more state than needed:

```java
public record OrderListRow(Long id, String status, BigDecimal total) {}

@Query("""
       select new com.example.api.OrderListRow(o.id, o.status, o.total)
       from Order o
       where o.customer.id = :customerId
       order by o.createdAt desc, o.id desc
       """)
Slice<OrderListRow> findOrderRows(Long customerId, Pageable pageable);
```

Benefits:

- exact columns and relationships;
- no accidental lazy traversal during mapping;
- smaller persistence-context load;
- clear read-only intent;
- less serialization coupling.

Interface projections can be concise, but verify nested projection behavior and generated SQL. DTO projections are often clearer for complex endpoints.

A projection does not automatically make a query fast. Inspect joins, indexes, row counts, and count queries.

---

## 9. EAGER Loading Pitfalls

EAGER can cause:

- unnecessary data for endpoints that do not need it;
- extra selects rather than one join;
- N+1 when the provider loads an eager relationship with separate selects for many roots instead of one join;
- larger memory use and wider SQL;
- difficult global behavior that cannot be easily disabled per use case.

EAGER is a mapping requirement, not a promise of an optimal join plan. Prefer explicit fetch plans and keep associations lazy unless the domain has a strong, measured reason otherwise.

---

## 10. Pagination and Fetch Plans

For a page of root entities:

```java
Page<Order> findByStatus(OrderStatus status, Pageable pageable);
```

A `Page` commonly performs a content query and a count query. Adding collection fetching can make both semantics and performance worse.

### Two-step collection loading

```text
1. select page of order IDs, stable order
2. select orders + required to-one/lines where id in (...)
3. map and restore page order
```

This is often more predictable for a page with collection data. The service must handle the second query and avoid accidentally returning a different order.

### Keyset pagination

For a large, append-like feed:

```sql
where (created_at, id) < (:lastCreatedAt, :lastId)
order by created_at desc, id desc
limit :size
```

Keyset avoids scanning large offsets and provides more stable traversal under inserts, but the client must carry a cursor and the ordering must be compatible with the cursor.

---

## 11. Fetch Plan Decision Table

| Requirement | Good starting point |
| --- | --- |
| One order with one customer | to-one fetch join or entity graph |
| Page orders with summary fields | DTO/projection query |
| Page orders with lines | two-step root-page + graph fetch |
| Many roots need one to-one relation | fetch join or batch fetching |
| Many roots need a collection but row multiplication is high | batch fetching or second query |
| API must know total pages | `Page`, inspect count query |
| API only needs “more available?” | `Slice` |
| Stable high-volume feed | keyset pagination |
| Lazy access after service returns | load/map inside transaction; do not rely on OSIV |

---

## 12. Scenario Reasoning

### Scenario A: N+1 mapper

A repository returns 100 orders. The mapper reads `order.getCustomer().getName()` for each.

**Answer:** One root query plus up to 100 customer queries. Use a customer fetch plan, batch fetching, or a DTO query according to response needs. Verify with SQL count.

### Scenario B: EAGER “fix”

A team changes all relationships to EAGER after a lazy-loading exception.

**Answer:** The exception may disappear while query count, memory, and latency become worse. Keep mappings deliberate and load the required graph inside the use-case transaction.

### Scenario C: collection fetch page

A page query fetch joins orders and lines and returns duplicate-looking orders.

**Answer:** The SQL result is multiplied by line rows, so pagination and count semantics are unsafe. Use a two-step query or a projection designed for the page.

### Scenario D: OSIV and serialization

The endpoint succeeds locally but production latency grows with order count.

**Answer:** Serialization may be triggering lazy queries under OSIV. Add request query-count metrics, inspect repeated SQL, and replace entity serialization with a DTO/fetch plan.

---

## 13. SDE-2 Interview Answers

### What is N+1?

One query loads N root records and then additional queries load related data per root, producing N+1 round trips. It is diagnosed by repeated parameterized SQL and query count growing with result size. Fixes include fetch joins, entity graphs, DTO queries, and batch fetching, chosen according to cardinality and pagination.

### Why not make everything EAGER?

EAGER is a global mapping requirement, not a per-use-case query plan. It can load unnecessary graphs, issue extra selects, create N+1, and increase memory. Explicit fetch plans are more predictable.

### Fetch join versus entity graph?

A fetch join expresses the join in JPQL and gives direct query-shape control. An entity graph specifies attributes to fetch around a repository query and can improve readability/reuse. Both still require analysis of collection multiplication and pagination.

### How do you fix a lazy-loading exception?

Identify the response data required, load it inside the service transaction using a fetch join/entity graph/DTO query or batch strategy, map to a DTO, and return after the persistence context no longer needs to be accessed. Do not blindly switch to EAGER or rely on OSIV.

---

## 14. Practice Batch

1. How would you prove an endpoint has N+1 rather than merely suspecting it?
2. Why is a collection fetch join risky with pagination?
3. When is batch fetching preferable to a fetch join?
4. What is the difference between `Page` and `Slice` in database work?
5. Design a fetch plan for an order-summary endpoint that does not need order lines.

### Model answer key

1. Correlate a request with SQL logs/metrics, count repeated parameterized queries, and compare query count as result size changes. One root query plus linearly growing relationship queries is strong evidence.
2. Collection rows multiply root rows, so offset/limit applies to joined rows rather than clean roots and can produce duplicate/missing roots or expensive in-memory processing.
3. When many roots need a relationship but joining would create excessive row multiplication, batch fetching groups deferred loads into fewer queries.
4. `Page` commonly requires a count query for total metadata; `Slice` can fetch one extra row or equivalent information to determine whether more results exist without total count.
5. Use a DTO projection selecting order id/status/total and stable sort fields, with indexes supporting customer/status/order columns. Do not load the entity graph or lines.

---

## Session Handoff

**Prepared study material:** Session 5 — Fetching and N+1  
**Topics covered:** lazy loading, `LazyInitializationException`, N+1 detection, fetch joins, entity graphs, batch fetching, DTO projections, EAGER pitfalls, pagination, keyset pagination, and endpoint fetch-plan design.  
**Next session:** Session 6 — Performance and Database Interaction  
**Next starting topic:** JDBC batching -> bulk DML -> persistence-context memory -> indexes/query plans/connection pools

## One-Line Revision Summary

Fetching is a use-case query plan: diagnose N+1 with SQL evidence, use joins/graphs/DTOs/batching according to cardinality, and treat pagination plus collection loading as a deliberate design problem.
