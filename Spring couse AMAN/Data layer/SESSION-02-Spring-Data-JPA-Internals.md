# Session 2 — Spring Data JPA Internals

**Track:** JPA / Hibernate SDE-2 Mastery  
**Scope:** repository abstractions → proxy → `SimpleJpaRepository` → `save()` → query derivation → `@Query` → pagination and sorting

## Session Outcome

By the end of this session, you should be able to:

- explain what `JpaRepository` adds over lower-level repository interfaces;
- describe how a repository interface call reaches a generated proxy;
- explain the role of `SimpleJpaRepository` without memorizing source-code trivia;
- trace `repository.save(entity)` through new-versus-existing detection;
- distinguish the original entity from the managed instance returned by `merge()`;
- reason about identifier strategies and `Persistable.isNew()`;
- choose between derived queries, JPQL, native SQL, and projections;
- explain how pagination and sorting affect the generated queries and API behavior;
- identify common Spring Data JPA bugs such as accidental `merge()`, unsafe derived names, count-query failures, and unbounded result loading.

This session assumes that `persist()` and `merge()` are already understood. The goal is to connect those JPA operations to the Spring Data repository API rather than re-teach their core lifecycle rules.

Before starting, read the [Foundation Bridge](JPA-SDE-2-Foundation-Bridge.md) sections on JPA/JDBC, `find()` versus `getReference()`, and query selection. Use the [Concept Map and Retention Plan](JPA-SDE-2-Concept-Map-and-Retention-Plan.md) to retrieve the mental model before reading the answer details.

---

## 1. The Abstraction Stack

Spring Data JPA does not replace JPA. It builds a repository programming model on top of an `EntityManager`.

```text
Application service
        |
        v
UserRepository interface
        |
        v
Spring Data generated proxy
        |
        +--> query-method implementation
        +--> @Query implementation
        +--> repository base class
                 |
                 v
          SimpleJpaRepository
                 |
                 v
             EntityManager
                 |
                 v
       JPA provider, usually Hibernate
                 |
                 v
          JDBC connection / database
```

The key idea is **method dispatch**, not magic persistence:

- a repository interface declares a contract;
- Spring Data inspects that contract at startup;
- it creates a proxy that handles method calls;
- standard CRUD calls are delegated to a repository implementation such as `SimpleJpaRepository`;
- query methods are parsed or backed by an explicit query;
- the implementation ultimately uses the `EntityManager`.

A repository is therefore a boundary around persistence operations, not a replacement for transaction design, fetch planning, or domain decisions.

---

## 2. Repository Interfaces and Capabilities

A typical repository might be:

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerIdAndStatus(Long customerId, OrderStatus status);
}
```

The type parameters mean:

- `Order`: the entity type;
- `Long`: the entity identifier type.

The interface hierarchy commonly looks like this:

```text
Repository<T, ID>
        |
        +--> CrudRepository<T, ID>
        |       basic CRUD and Iterable results
        |
        +--> PagingAndSortingRepository<T, ID>
        |       paging and sorting methods
        |
        +--> ListCrudRepository<T, ID>        [newer Spring Data lines]
        |
        +--> JpaRepository<T, ID>
                JPA-specific repository operations,
                List-oriented CRUD, flush-related methods,
                batch delete methods, and more
```

Exact inherited methods vary by Spring Data version. The interview-level point is stable: `JpaRepository` is an API contract; the runtime implementation and proxy are supplied by Spring Data.

### Why not inject `SimpleJpaRepository` directly?

The interface is the application-facing contract. Directly depending on the implementation:

- exposes framework details to business code;
- makes replacement or customization harder;
- reduces the ability to define a narrow repository API;
- does not solve transaction or query-design problems.

Prefer a focused interface:

```java
public interface InventoryRepository extends JpaRepository<InventoryItem, Long> {
    Optional<InventoryItem> findBySku(String sku);

    boolean existsBySku(String sku);
}
```

A focused repository can still inherit broad methods, but the service should use the smallest meaningful operation and avoid exposing persistence entities unnecessarily at the API boundary.

---

## 3. How the Repository Proxy Works

At application startup, Spring Data reads repository metadata and creates a proxy for the interface. The proxy can route different methods through different paths:

| Method shape | Typical handling |
| --- | --- |
| `save(entity)` | Base repository implementation |
| `findById(id)` | Base repository implementation using `EntityManager.find()` |
| `delete(entity)` | Base repository implementation using managed-state rules |
| `findByEmail(...)` | Query derivation parser creates a query |
| Method with `@Query` | Declared query is used |
| Custom fragment method | Application-provided fragment implementation |
| Paging method | Query plus count-query / page metadata path |

The proxy is not a transaction by itself. In a normal service-oriented design, the transaction boundary belongs on the service use case:

```java
@Service
public class OrderService {
    private final OrderRepository orders;

    @Transactional
    public void confirm(Long orderId) {
        Order order = orders.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));
        order.confirm(); // managed entity; dirty checking can persist this
    }
}
```

The repository call obtains or manipulates persistence state. The service transaction determines whether that work has a consistent unit-of-work boundary.

### A common misconception

> “Every repository method opens and commits its own transaction.”

That is not a safe general model. Transaction behavior comes from the surrounding Spring configuration and call path. A repository method may run inside an existing transaction, use a repository-level transactional setting, or fail when a modifying operation requires a transaction and none exists. Always identify the actual proxy and transaction boundary.

---

## 4. `SimpleJpaRepository`: The Standard CRUD Implementation

`SimpleJpaRepository` is Spring Data JPA's standard implementation for many repository operations. It is usually created behind the repository proxy and receives an `EntityManager` plus entity metadata.

Conceptually:

```java
public class SimpleJpaRepository<T, ID> {
    private final EntityManager entityManager;
    private final JpaEntityInformation<T, ID> entityInformation;

    public <S extends T> S save(S entity) {
        if (entityInformation.isNew(entity)) {
            entityManager.persist(entity);
            return entity;
        }
        return entityManager.merge(entity);
    }
}
```

This is a teaching model, not a promise that every framework version has exactly this source layout. The important behavior is:

1. determine whether the entity is new;
2. call `persist()` for a new entity;
3. call `merge()` for an entity considered existing;
4. return the object that the caller should use afterward.

`SimpleJpaRepository` also implements operations such as `findById`, `existsById`, deletes, query execution, flush, and paging. Framework source versions differ, so understand the contract and delegation flow rather than memorizing private helper names.

---

## 5. `save()` End-to-End

### The basic flow

```java
Order saved = orderRepository.save(order);
```

Reason through it as:

```text
service calls repository.save(order)
        |
        v
repository proxy dispatches save()
        |
        v
repository implementation asks: is this entity new?
        |
        +--> yes: EntityManager.persist(order)
        |          return the same object
        |
        +--> no:  EntityManager.merge(order)
                   return the managed instance from merge()
```

The operation normally does not mean “execute an SQL `INSERT` or `UPDATE` right now.” It means “associate the state with the persistence context according to JPA semantics.” SQL timing is controlled by flush and the transaction/provider behavior.

### New entity path

```java
Order order = new Order();
order.setStatus(OrderStatus.CREATED);

Order saved = orderRepository.save(order);

assert saved == order; // conceptual behavior of persist path
```

`persist()` makes the original instance managed. Dirty checking then follows that object. The `INSERT` may be emitted during flush rather than at the exact line containing `save()`.

### Existing entity path

```java
Order detached = loadFromRequest();
Order saved = orderRepository.save(detached);

// saved is the object to use for managed-state changes.
// detached is not made managed merely because save() was called.
saved.setStatus(OrderStatus.CONFIRMED);
```

For an existing entity, Spring Data commonly uses `merge()`. `merge()` copies state into a managed instance and returns that managed instance. The supplied `detached` object remains detached.

### The most important `save()` rule

> Always use the return value of `save()` when the save path may call `merge()`.

This is especially important when code modifies the object after saving:

```java
Order input = requestMapper.toEntity(request);
Order managed = orderRepository.save(input);
managed.setStatus(OrderStatus.CONFIRMED); // tracked if managed
input.setStatus(OrderStatus.CANCELLED);   // may not be tracked
```

A service that ignores the return value can silently mutate a detached object and expect dirty checking to persist it.

---

## 6. How Spring Data Decides `isNew`

The new-versus-existing decision is not simply “does the database contain this row?” It is an entity-metadata decision made before the repository chooses `persist()` or `merge()`.

Common strategy, depending on entity metadata and Spring Data version:

1. If the entity implements `Persistable`, use `isNew()`.
2. Otherwise, inspect a non-primitive version property if one exists.
3. Otherwise, inspect the identifier property; a null identifier usually indicates new state.
4. Primitive identifiers cannot use `null`, so their default-value behavior needs care.

The exact details are framework-version-sensitive, but the design question is stable: **Can the repository reliably distinguish an entity that has never been persisted from one that already exists?**

### Generated identifier example

```java
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```

A new `Order` usually has `id == null`, so it is commonly treated as new and passed to `persist()`.

### Assigned identifier problem

```java
@Entity
public class ExternalPayment {
    @Id
    private String providerReference;
}
```

A new object can have a non-null assigned identifier. A naive null-id rule can classify it as existing and route it through `merge()`, which changes lifecycle behavior and can cause an unexpected `SELECT` or an update/insert decision different from what the application intended.

Possible solutions include:

- implement `Persistable<ID>` with a correct `isNew()` lifecycle strategy;
- use a nullable version property when appropriate;
- keep construction and persistence state explicit;
- avoid treating an identifier's presence as proof that a row exists.

### `Persistable` warning

`isNew()` is application-controlled. If it becomes stale after persistence, the repository may choose the wrong operation. A robust implementation commonly flips the new flag through persistence callbacks, but callback design must account for rehydration and detached objects.

Do not use `Persistable` to hide a broken aggregate lifecycle. Test new, reloaded, detached, and merged cases separately.

---

## 7. `save()` Does Not Mean “Always Update”

The word `save` is intentionally broad. Depending on state, it may lead to:

| Input situation | Typical JPA operation | Managed object after call | Possible SQL at flush |
| --- | --- | --- | --- |
| New entity, recognized as new | `persist(entity)` | Original object | `INSERT` |
| Existing/detached entity | `merge(entity)` | Returned managed instance | `UPDATE`, sometimes insert-like behavior for new merge state |
| Already managed entity | Often still routed through save logic | Existing managed instance or merge result | Dirty checking determines whether SQL is needed |

Calling `save()` for an already managed entity is often unnecessary. In a transactional service, loading an entity and mutating it is normally enough:

```java
@Transactional
public void changeStatus(Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    order.setStatus(OrderStatus.CONFIRMED); // dirty checking
}
```

Adding `save(order)` may be harmless in a common path, but it can obscure the lifecycle model and is not a substitute for a transaction.

### `saveAndFlush()`

`saveAndFlush()` combines save behavior with an explicit flush. It still does not mean transaction commit. Use it when the current unit of work genuinely needs database synchronization before continuing, such as:

- a constraint or trigger result must be observed before another operation;
- a generated database value must be available at a known point;
- a test or boundary requires explicit synchronization.

Do not use it as a reflexive “make persistence reliable” call. Frequent flushes can increase round trips and reduce batching.

---

## 8. Query Methods: Derived Queries

Spring Data can derive a query from a method name:

```java
Optional<Order> findByCustomerIdAndStatus(
        Long customerId,
        OrderStatus status
);

List<Order> findTop20ByStatusOrderByCreatedAtDesc(OrderStatus status);

boolean existsByCustomerIdAndStatus(Long customerId, OrderStatus status);
```

The parser identifies:

- subject: `find`, `exists`, `count`, `delete`, `remove`, `top`, `first`;
- predicate: properties and operators after `By`;
- conjunction/disjunction: `And`, `Or`;
- comparisons: `LessThan`, `Between`, `In`, `Containing`, `StartingWith`, and others;
- ordering: `OrderByCreatedAtDesc`;
- limiting: `Top20` or `First20`.

### Property traversal

```java
List<Order> findByCustomer_Address_CityIgnoreCase(String city);
```

This may traverse `order.customer.address.city`, but long method names create coupling to the entity model and can become difficult to review.

### When derivation is a good fit

- short, stable predicates;
- straightforward equality and range conditions;
- methods whose name remains readable;
- simple existence/count operations.

### When to prefer an explicit query

- the method name becomes a paragraph;
- grouping, joins, conditional expressions, or subqueries matter;
- fetch strategy needs to be deliberate;
- the query is part of a performance-sensitive path;
- an API projection is clearer than returning a full entity.

The method name is part of the persistence contract. A renamed property can break application startup, so treat derived methods as compile/startup-sensitive code, not casual text.

---

## 9. `@Query`: JPQL versus Native SQL

### JPQL

JPQL queries entities and their mapped attributes, not tables and raw columns:

```java
@Query("""
       select o
       from Order o
       where o.customer.id = :customerId
         and o.status = :status
       order by o.createdAt desc
       """)
List<Order> findRecentOrders(
        @Param("customerId") Long customerId,
        @Param("status") OrderStatus status
);
```

JPQL is portable across JPA providers and follows the object model. It is a good default for entity-oriented queries.

### Native SQL

```java
@Query(value = """
       select *
       from orders
       where customer_id = :customerId
       order by created_at desc
       """, nativeQuery = true)
List<Order> findRecentOrdersNative(@Param("customerId") Long customerId);
```

Native SQL is database-language SQL. It is useful for database-specific features, complex SQL, tuned plans, reporting queries, and features not expressed well in JPQL. It increases coupling to schema and database dialect and can require explicit count queries for pagination.

### Comparison

| Concern | JPQL | Native SQL |
| --- | --- | --- |
| Query vocabulary | Entities and attributes | Tables and columns |
| Portability | Higher | Lower |
| Mapping alignment | Entity model | Database schema |
| Database-specific features | Limited | Strong |
| Refactoring support | Better for entity fields | Schema changes must be handled explicitly |
| Performance | Not automatically worse | Not automatically better; inspect SQL and plan |

Neither JPQL nor native SQL guarantees good performance. Measure generated SQL, cardinality, indexes, joins, and execution plans.

### Parameter binding

Use named parameters instead of string concatenation:

```java
@Query("select o from Order o where o.status = :status")
List<Order> findByStatus(@Param("status") OrderStatus status);
```

Binding protects query structure and makes intent clearer. Dynamic sorting, filtering, and optional predicates need deliberate design; do not concatenate untrusted values into query text.

---

## 10. Modifying Queries

A query that changes data needs different semantics from a read query:

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("""
       update Order o
       set o.status = :newStatus
       where o.customer.id = :customerId
       """)
int updateCustomerOrderStatus(
        @Param("customerId") Long customerId,
        @Param("newStatus") OrderStatus newStatus
);
```

Important consequences:

- bulk JPQL updates bypass normal per-entity dirty checking;
- already-managed entities can hold stale state after the bulk operation;
- the operation normally needs a transaction;
- clearing the persistence context can prevent stale managed objects from being reused;
- the returned count is affected rows, not necessarily entity count in every conceptual sense.

The safest mental model is:

```text
bulk update/delete -> database rows change directly
                   -> managed entity snapshots are not individually updated
                   -> persistence context may now be stale
```

Use bulk operations for the right workload, then define how stale state is handled. Do not mix them casually with code that continues mutating old managed instances.

---

## 11. Pagination and Sorting

### Page versus Slice versus List

```java
Page<Order> findByStatus(OrderStatus status, Pageable pageable);

Slice<Order> findByCustomerId(Long customerId, Pageable pageable);

List<Order> findByStatus(OrderStatus status, Pageable pageable);
```

Conceptually:

- `List`: result rows only; no total-count metadata;
- `Slice`: current content plus whether another slice exists; can avoid a total count;
- `Page`: content plus total elements/pages, commonly requiring a count query.

A `Page` commonly triggers two queries:

1. content query with limit/offset;
2. count query for total metadata.

That count query can be expensive on large or complex datasets.

### Sorting

```java
Pageable pageable = PageRequest.of(
        0,
        50,
        Sort.by(Sort.Direction.DESC, "createdAt")
                .and(Sort.by(Sort.Direction.ASC, "id"))
);
```

A stable tie-breaker such as `id` is important when records share the same primary sort value. Without deterministic ordering, records can move between pages as the database returns ties in arbitrary order.

### Offset pagination limitations

Offset pagination is easy to use but can become slower at large offsets and can produce duplicates/misses when rows are inserted or deleted between requests. For high-volume feeds, consider keyset/seek pagination:

```text
where (created_at, id) < (:lastCreatedAt, :lastId)
order by created_at desc, id desc
limit :pageSize
```

The exact implementation depends on database and framework support. The SDE-2 design point is to match pagination strategy to access pattern and consistency requirements.

### Fetch joins and pagination

A collection fetch join combined with pagination can multiply rows and make page boundaries ambiguous. Hibernate may warn, de-duplicate in memory, or produce unexpectedly expensive behavior depending on version and configuration. A safer pattern is often:

1. page over root IDs;
2. fetch the required graph in a second query using those IDs;
3. restore the requested order.

This connects Session 2 query APIs to the deeper fetching topics in Session 5.

---

## 12. Code-Trace Scenarios

### Scenario A: ignoring the merge result

```java
@Transactional
public void confirm(OrderDto dto) {
    Order input = mapper.toEntity(dto); // existing id; detached-style object
    orderRepository.save(input);        // likely merge path
    input.setStatus(OrderStatus.CONFIRMED);
}
```

**Reasoning:** `save(input)` may return a managed instance from `merge()`, while `input` remains detached. The later mutation can be lost to dirty checking. Use:

```java
Order managed = orderRepository.save(input);
managed.setStatus(OrderStatus.CONFIRMED);
```

Better still, when the use case is an update, load the managed entity inside a transaction, authorize it, apply the command, and let dirty checking persist the change.

### Scenario B: new entity with assigned identifier

```java
ExternalPayment payment = new ExternalPayment("provider-123");
payment.setAmount(amount);
paymentRepository.save(payment);
```

**Reasoning:** A non-null assigned identifier does not prove that a row already exists. Verify the entity's new-state strategy. If it is misclassified, `merge()` may perform a lookup and the behavior may differ from the intended insert path.

### Scenario C: `Page` surprises

```java
Page<Order> page = repository.findByStatus(status,
        PageRequest.of(0, 100));
```

**Reasoning:** Expect a content query and commonly a count query. Inspect both. A fast content query does not mean the endpoint is fast if the count query scans a large or complex result.

### Scenario D: bulk update followed by managed read

```java
Order order = repository.findById(id).orElseThrow();
repository.markAllPendingAsExpired(now);
String status = order.getStatus();
```

**Reasoning:** The bulk update bypasses the managed entity instance. `order` may still show stale status. Flush/clear or a deliberate reload strategy is required, depending on the repository annotation and transaction flow.

---

## 13. SDE-2 Interview Answers

### What happens when `repository.save(entity)` is called?

Spring Data dispatches the repository call through a generated proxy to its repository implementation. The implementation determines whether the entity is new. For a new entity it normally calls `EntityManager.persist()` and returns the same instance. For an existing entity it normally calls `EntityManager.merge()` and returns the managed instance produced by merge. SQL is generally synchronized at flush, and commit determines transaction durability.

### Why does `save()` return an entity?

Because the existing-entity path may use `merge()`, and the managed instance returned by `merge()` is the instance whose changes dirty checking follows. Ignoring the return value can lead to mutating a detached object and seeing no update.

### How does Spring Data know whether an entity is new?

It uses entity metadata and, where applicable, `Persistable.isNew()`. Without custom new-state handling, identifier and version-property conventions are commonly used. A non-null identifier is not universally proof that the entity exists, especially with assigned IDs.

### Is `save()` required after changing a managed entity?

Usually not inside a transaction. If the entity was loaded into the current persistence context, changing it is enough for dirty checking. `save()` may make intent explicit or support a detached input flow, but it does not replace a transaction and can obscure whether the object is managed.

### Is native SQL faster than JPQL?

Not inherently. Performance depends on generated SQL, indexes, joins, data distribution, database plan, network cost, and result mapping. Native SQL is valuable for database-specific or complex queries, but choose it for a justified reason and measure it.

### Why can a page execute two queries?

A `Page` usually needs the content query and a count query to calculate total elements/pages. A `Slice` can avoid the total count when the client only needs to know whether more data exists.

---

## 14. Practice Batch: Answer Before Reading the Key

1. A service calls `save(detachedEntity)` and then changes `detachedEntity`. Why might no update occur?
2. Why can an assigned identifier confuse new-entity detection?
3. What is the difference between `Page`, `Slice`, and `List` for a repository method?
4. When would you choose a derived query, JPQL, or native SQL?
5. Why can a bulk update make an already-loaded managed entity stale?

### Model answer key

1. The existing path may call `merge()`, which returns a managed instance while leaving the supplied object detached. The later mutation must be applied to the returned managed object or to an entity loaded in the transaction.
2. A new object can have a non-null identifier before any database row exists. Null-ID heuristics may classify it as existing and route it to `merge()`.
3. `List` returns content only, `Slice` provides content plus whether another slice exists, and `Page` adds total-count metadata and commonly requires a count query.
4. Use derived queries for short stable predicates, JPQL for expressive entity-oriented queries, and native SQL for justified database-specific or complex SQL needs. Measure performance rather than assuming one is faster.
5. Bulk DML changes database rows directly and bypasses per-entity dirty checking. The persistence context does not automatically update every managed object or snapshot, so a reload/clear strategy may be needed.

---

## 15. Design Exercise: Build a Repository Boundary

Design a repository for an order search screen with these requirements:

- search by customer and status;
- show only a summary, not the entire aggregate;
- sort newest first with stable ordering;
- return 25 items per request;
- allow the caller to know whether another page exists;
- avoid loading all order lines.

A reasonable starting contract is:

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    Slice<OrderSummary> findByCustomerIdAndStatusOrderByCreatedAtDescIdDesc(
            Long customerId,
            OrderStatus status,
            Pageable pageable
    );
}
```

Then question the design:

- Is interface projection sufficient, or is a DTO query clearer?
- Does `Slice` meet the UI requirement, or is total count needed?
- Is the ordering stable under equal timestamps?
- Does the query fetch only summary columns?
- Are `customerId` and `status` indexed appropriately?
- Does the service map the result without triggering lazy loads?

There is no single correct repository signature. The SDE-2 answer explains the trade-offs and validates them with SQL and query plans.

---

## 16. Common Incorrect Statements

- “`save()` always executes an `INSERT` or `UPDATE` immediately.”
- “A non-null ID means the row already exists.”
- “`save()` makes the object passed to it managed in every case.”
- “Calling `save()` is mandatory after every field change.”
- “Native SQL is always faster than JPQL.”
- “A `Page` is just a `List` with metadata and costs the same.”
- “Derived query methods are harmless because the compiler checks their property names.”
- “Bulk updates update all loaded entity objects automatically.”

More precise replacements:

- “`save()` chooses a JPA persistence operation; flush controls synchronization timing.”
- “New-state detection depends on metadata and configuration; assigned IDs require care.”
- “Use the returned object when the path may call `merge()`.”
- “Managed entities inside a transaction usually rely on dirty checking.”
- “Native SQL is a tool for justified database-specific or complex queries, not a performance guarantee.”
- “`Page` commonly adds a count query.”
- “Derived query methods are parsed at startup and can fail due to model or naming changes.”
- “Bulk DML can leave the persistence context stale.”

---

## Session Handoff

**Prepared study material:** Session 2 — Spring Data JPA Internals  
**Topics covered:** repository abstractions, generated proxy routing, `SimpleJpaRepository`, `save()` flow, new-entity detection, `Persistable`, derived queries, `@Query`, JPQL/native SQL, modifying queries, pagination, sorting, and code scenarios.  
**Next live-study checkpoint:** Begin with `JpaRepository` and trace one `save()` call before moving to query derivation.  
**Do not restart:** `merge()` fundamentals; use them only to explain the `save()` existing-entity path.  
**Next session:** Session 3 — Entity Relationships  
**Next starting topic:** `@ManyToOne` → `@OneToMany` → owning side and `mappedBy`

## One-Line Revision Summary

Spring Data repositories are proxies over JPA operations: `save()` usually chooses `persist()` for new state and `merge()` for existing state, while query methods compile into entity or SQL queries whose transaction, fetch, pagination, and indexing choices still determine production behavior.
