# JPA SDE-2 Foundation Bridge

**Purpose:** Fill the small but important gaps between the JPA fundamentals checklist and Session 2.  
**Use:** Read once before or alongside Session 2, then revisit only the sections exposed by the review questions.  
**Roadmap impact:** This is a bridge, not a new session and not a replacement for the fixed 12-session plan.

## The Five Questions for Any Persistence Concept

For every API, mapping, or performance feature, answer these in order:

1. **What problem does it solve?**
2. **Which layer owns the behavior?** Domain model, JPA contract, Hibernate provider, JDBC, or database?
3. **What object/state is affected?** NEW, MANAGED, DETACHED, REMOVED, query result, cache entry, or row lock?
4. **When can SQL happen?** At the method call, flush, commit, or a later explicit query?
5. **What can go wrong in production?** Stale state, extra queries, wrong transaction, memory growth, lock contention, or portability?

This prevents annotation memorization from replacing a mental model.

---

## 1. JPA, Hibernate, JDBC, and the Database

These names describe different layers:

```text
Application service
    -> Spring Data repository API
    -> JPA API and contract: EntityManager, JPQL, mappings
    -> Hibernate provider: Session, dirty checking, SQL generation
    -> JDBC API/driver: Connection, PreparedStatement, ResultSet
    -> Database: tables, indexes, locks, plans, transactions
```

### JPA

JPA is the persistence specification and programming model. It defines concepts such as entities, persistence contexts, lifecycle operations, relationships, JPQL, and `EntityManager`. It does not itself execute database calls; a provider implements it.

### Hibernate

Hibernate is a JPA provider and an ORM implementation. It translates entity operations and JPQL into SQL, manages snapshots/proxies/action queues, and provides additional native features. Hibernate can be used through JPA APIs or directly through its `Session` API.

### JDBC

JDBC is the lower-level Java database API. It works with SQL, connections, prepared statements, result sets, transaction commits, and rollback. JDBC does not give you entity identity, dirty checking, relationship mappings, or automatic state synchronization.

| Need | JPA/Hibernate | JDBC |
| --- | --- | --- |
| Map rows to an object graph | Strong support | Application code does it |
| Dirty checking | Yes for managed entities | No |
| Relationship/lifecycle cascades | Yes | No automatic equivalent |
| SQL control | Indirect unless native query | Direct |
| Portability across databases | Higher at the JPA level | SQL/schema dependent |
| Bulk/reporting/tuned SQL | Sometimes less direct | Often natural |
| Main cost | Hidden queries and ORM complexity | Repeated mapping and manual transaction code |

### SDE-2 decision rule

Use JPA for aggregate-oriented transactional work when identity, relationships, and lifecycle behavior help. Use JDBC/native SQL for set-based reporting, database-specific operations, or paths where explicit SQL control is worth the coupling. The choice is about workload and invariants, not which technology is universally faster.

---

## 2. Entity Mapping Basics

A minimal entity:

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @Column(nullable = false)
    private String status;

    protected Order() {
        // Required by JPA providers for reflective construction.
    }

    public Order(String status) {
        this.status = status;
    }
}
```

### `@Entity`

Marks a class as persistent entity type. It participates in the persistence context and is mapped to a table by default or by `@Table`. An entity is not automatically managed just because its class has `@Entity`; it becomes managed through operations such as `find()`, `persist()`, or the managed result of `merge()`.

### `@Id`

Defines the entity identity used by the persistence context and database mapping. Identity is not the same as business equality. A database primary key is usually the safest stable identity for persistence operations, while business keys may have separate uniqueness constraints.

### `@GeneratedValue`

Delegates identifier generation to a provider/database strategy. Common strategies:

- `IDENTITY`: database-generated identity value; can require an insert before the ID is known and may limit batching.
- `SEQUENCE`: sequence-backed values; allocation can support batching and reduce round trips, depending on database/provider configuration.
- `TABLE`: table-based generation; portable in concept but usually more coordination overhead.
- `AUTO`: provider chooses a strategy; convenient but less explicit across databases.

The strategy affects insert timing, batching, migrations, and portability. Do not choose it only because the annotation is familiar.

### Constructors and access

A JPA provider needs a no-argument constructor, commonly `protected` or `public`. Keep it available for rehydration and use domain constructors/factory methods for valid new objects.

Field access and property access determine where the provider reads mapped state. Keep mapping annotations and lifecycle assumptions consistent with the chosen access strategy. Avoid putting business logic in setters merely because a provider may call them during hydration.

### Schema constraints are part of the mapping decision

`@Column(nullable = false)` expresses a mapping constraint, but production schema should be managed by an explicit migration and verified in the database. Use database constraints for non-null, uniqueness, foreign-key, and check rules that must hold under concurrent writers.

---

## 3. `find()` versus `getReference()`

Both can return a managed representation of an existing identity, but they communicate different loading needs.

| Operation | Initial database access | Missing row behavior | Best mental model |
| --- | --- | --- | --- |
| `find(Entity.class, id)` | Usually loads state immediately, or returns the context instance | Returns `null` | “Read this entity now.” |
| `getReference(Entity.class, id)` | May return a lazy proxy without loading all state immediately | Failure commonly appears when state is accessed or during synchronization | “I need a managed reference for this identity.” |

Example:

```java
Order order = entityManager.find(Order.class, id);
if (order == null) {
    throw new OrderNotFoundException(id);
}

Order reference = entityManager.getReference(Order.class, id);
// The provider may not select all columns yet.
```

### Use `find()` when

- you need to inspect fields;
- not-found should be handled immediately;
- authorization or business validation depends on current state;
- you want a clear read path.

### Use `getReference()` when

- you only need a managed identity to set a foreign-key relationship;
- you are deleting or associating an entity and do not need its fields first;
- avoiding an unnecessary full select is valuable and the provider/database behavior is verified.

```java
Order order = entityManager.getReference(Order.class, orderId);
Shipment shipment = new Shipment(order);
entityManager.persist(shipment);
```

### Important caveats

- `getReference()` is not a not-found check. Accessing a missing proxy can raise an `EntityNotFoundException` rather than returning `null`.
- If the persistence context already contains the entity, identity-map behavior can return that managed instance.
- Accessing any unloaded field can trigger SQL while the context is active.
- A reference is not a permission check. Authorization still needs a deliberate query/state decision.
- In a repository API, `getReferenceById()` has the same conceptual distinction; do not use it when the caller expects an immediate existence check.

### One-line memory hook

`find()` answers **“what is there?”**; `getReference()` supplies **“the identity I intend to use.”**

---

## 4. Spring Data Query Selection Ladder

Choose the least complex tool that expresses the real query:

```text
short stable predicate
    -> derived query method

entity-oriented query with joins/fetch/grouping
    -> @Query with JPQL

optional filters assembled at runtime
    -> Specification / Criteria API

narrow read model or aggregation
    -> interface projection or DTO projection

database-specific/reporting/tuned SQL
    -> native query or JDBC
```

This is a decision ladder, not a performance ranking. Every option still needs SQL, index, cardinality, and transaction analysis.

### Derived query

```java
List<Order> findByCustomerIdAndStatus(
        Long customerId,
        OrderStatus status
);
```

Best for short readable predicates. Long names couple query structure to entity property names and become difficult to review.

### JPQL with `@Query`

```java
@Query("""
       select o from Order o
       where o.customer.id = :customerId
         and o.status in :statuses
       """)
List<Order> search(
        Long customerId,
        Collection<OrderStatus> statuses
);
```

Best when the entity model expresses the query clearly and the query deserves explicit review.

### Native query

Use when database-specific syntax, a tuned plan, reporting SQL, or a feature unavailable in JPQL justifies schema coupling. Native SQL is not automatically faster.

---

## 5. Specifications and Criteria API

### The problem they solve

A search screen may have optional filters:

- customer is optional;
- status can contain several values;
- created-at range is optional;
- text search is optional;
- only active records should be returned.

Concatenating query strings is hard to test and unsafe. A single derived method becomes unreadable. Specifications compose predicates:

```java
public interface OrderRepository
        extends JpaRepository<Order, Long>,
                JpaSpecificationExecutor<Order> {
}
```

```java
Specification<Order> byCustomer(Long customerId) {
    return (root, query, builder) ->
            customerId == null
                    ? null
                    : builder.equal(root.get("customer").get("id"), customerId);
}

Specification<Order> byStatus(OrderStatus status) {
    return (root, query, builder) ->
            status == null
                    ? null
                    : builder.equal(root.get("status"), status);
}

Specification<Order> specification = Specification
        .where(byCustomer(customerId))
        .and(byStatus(status));

Page<Order> result = orderRepository.findAll(specification, pageable);
```

A null predicate means “no restriction” in the composed specification model. Keep the examples aligned with the Spring Data version used by the project; APIs around nullability and composition can evolve.

### Criteria API

Criteria API is the JPA programmatic query API underneath the same general idea:

```java
CriteriaBuilder builder = entityManager.getCriteriaBuilder();
CriteriaQuery<Order> query = builder.createQuery(Order.class);
Root<Order> order = query.from(Order.class);
query.select(order)
     .where(builder.equal(order.get("status"), status));
```

Use it when query structure must be built programmatically and provider portability matters. It is type-safe only when used with the generated static metamodel; string-based paths can still break at runtime.

### Trade-offs

| Tool | Strength | Cost |
| --- | --- | --- |
| Derived method | Fast to write/read when short | Name explosion |
| JPQL | Explicit entity query and joins | Dynamic optional predicates need design |
| Specification | Composable filters and paging | Can become opaque without named building blocks |
| Criteria API | Programmatic construction | Verbose; harder to read |
| Native/JDBC | Maximum SQL control | Schema/database coupling and manual mapping |

Do not introduce Specifications or Criteria just to appear flexible. Use them when the query genuinely has optional composition or dynamic structure.

---

## 6. Projections: Entity, Interface, DTO

### Entity result

Use when the use case needs managed state, relationship navigation, or domain behavior. It can load more columns/state than a read-only list requires.

### Interface projection

```java
public interface OrderSummaryView {
    Long getId();
    OrderStatus getStatus();
    BigDecimal getTotal();
}

List<OrderSummaryView> findByCustomerId(Long customerId);
```

Spring Data can map selected properties to an interface view. It is concise, but nested access and generated SQL should be verified. It is read-oriented and should not be treated as a managed entity.

### DTO/record projection

```java
public record OrderSummary(
        Long id,
        OrderStatus status,
        BigDecimal total
) {}

@Query("""
       select new com.example.api.OrderSummary(o.id, o.status, o.total)
       from Order o
       where o.customer.id = :customerId
       """)
List<OrderSummary> findSummaries(Long customerId);
```

DTO projections make the response shape explicit and are useful for aggregation, API boundaries, and preventing lazy traversal.

### Projection decision

```text
Need to mutate/domain-behave? -> entity
Need narrow stable read shape? -> DTO/record
Need concise simple read view? -> interface projection
Need aggregate/database-specific result? -> DTO/native/JDBC as justified
```

A projection reduces selected data, but indexes and joins still determine performance.

---

## 7. Repository Write Method Semantics

Spring Data exposes several methods whose names sound similar but have different lifecycle/performance behavior:

| Method | Core behavior | Main caution |
| --- | --- | --- |
| `save` | New detection -> `persist` or `merge` | Use returned object when merge is possible |
| `saveAll` | Repeats save semantics for an iterable | Not automatically one SQL batch or one safe transaction |
| `saveAndFlush` | Save, then explicit flush | Flush is not commit; can reduce batching |
| `delete(entity)` | Remove an entity according to managed-state rules | Entity state/transaction still matter |
| `deleteById(id)` | Delete by identifier through repository behavior | Verify select/delete SQL and missing-row behavior |
| `deleteAll` | Entity-oriented deletion | May load entities and honor lifecycle behavior |
| `deleteAllInBatch` | Bulk-style deletion | Can bypass callbacks and leave context/cache stale |

### `saveAll` is not a performance promise

`saveAll` often loops over entities and delegates to `save`. Throughput still depends on transaction scope, ID strategy, JDBC batching, flush frequency, and persistence-context size. For large imports, use an explicit bounded batch design and measure SQL.

### Entity-oriented delete versus bulk delete

Use entity-oriented operations when entity callbacks, cascades, orphan handling, authorization, or domain rules matter. Use bulk operations when set-based deletion is intended and stale persistence-context/cache behavior is handled explicitly.

---

## 8. Missing Topics: Priority and Ownership

| Topic | Status in current roadmap | Where to study |
| --- | --- | --- |
| JPA vs JDBC | Not explicit | This bridge, Section 1 |
| `@Entity`, `@Id`, `@GeneratedValue` | Examples exist but no focused mental model | This bridge, Section 2 |
| `find()` vs `getReference()` | Missing | This bridge, Section 3 |
| Specifications | Missing | This bridge, Section 5 |
| Criteria API | Missing | This bridge, Section 5 |
| Interface projections | Mentioned, not explained | This bridge, Section 6 |
| DTO projections | Covered later, now compared here | This bridge, Section 6 and Session 5 |
| `saveAll` and delete variants | Missing from the session plan | This bridge, Section 7 and Session 6 for batching |
| Flush modes | Conceptual in Session 7, needs a live decision exercise | Session 7 Section 7 plus retention review |
| OSIV | Covered in Session 4, needs live scenario practice | Session 4 and Session 10 |
| Entity callbacks, auditing, and attribute converters | Outside the fixed 12-session core | Optional follow-up after Session 12 |
| Bean Validation and persistence-boundary validation | Outside the fixed 12-session core | Optional follow-up after Session 12 |
| Spring Data tests, Testcontainers, and schema migrations | Production extensions, not current core | Optional follow-up after Session 12 |
| Soft delete, multi-tenancy, inheritance, and embeddables | Production extensions, not current core | Optional follow-up after Session 12 |

The last four are not silently forgotten. They are explicitly classified as follow-up topics so the core roadmap can finish without expanding indefinitely.

---

## 9. Retrieval Check: Answer Without Looking Back

1. What does JPA provide that JDBC does not?
2. Why does a JPA entity need a no-argument constructor?
3. When is `getReference()` a better fit than `find()`?
4. What happens if the referenced row does not exist?
5. When should you use a Specification instead of a derived method?
6. Is Criteria API automatically type-safe?
7. Is an interface projection a managed entity?
8. Why is `saveAll()` not automatically a JDBC batch?
9. Why can `deleteAllInBatch()` leave stale state?
10. What is the five-question lens for any persistence concept?

### Answer key

1. JPA provides entity identity, persistence-context tracking, dirty checking, mappings, relationships, and lifecycle semantics; JDBC provides direct SQL/connection primitives.
2. Providers commonly need one for reflective construction when rehydrating database state.
3. When only a managed identity is needed and loading all fields immediately is unnecessary, such as associating a foreign key. Use `find()` for immediate state/not-found handling.
4. A reference may fail when initialized or synchronized, commonly with `EntityNotFoundException`; it does not return `null` like `find()`.
5. When filters are optional/composable and one derived method or static JPQL becomes unwieldy.
6. Only with generated metamodel/types; string paths can still fail at runtime.
7. No. It is a read projection/view, not the managed entity instance to mutate.
8. It generally delegates save behavior across elements; transaction, ID strategy, flush, batching, and context size still control performance.
9. Bulk deletion can bypass per-entity tracking/callbacks, leaving managed objects and caches stale.
10. Problem, owning layer, affected state/object, SQL timing, and production failure mode.

---

## Bridge Handoff

**Use before/alongside:** Session 2 — Spring Data JPA Internals  
**Newly covered:** JPA vs JDBC, entity mapping basics, `find()` vs `getReference()`, Specifications, Criteria API, interface/DTO projections, `saveAll`, and delete method semantics.  
**Next live session remains:** Session 2 — start with `JpaRepository` and trace `save()` end-to-end.  
**Do not restart:** lifecycle fundamentals; use this bridge only to close the explicit coverage gaps.

## One-Line Revision Summary

JPA manages entity identity and lifecycle above JDBC; choose `find` versus `getReference` by loading intent, choose query tools by dynamism and result shape, and never confuse a repository method name with a batching, transaction, or consistency guarantee.
