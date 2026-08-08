# Session 3 — Entity Relationships

**Track:** JPA / Hibernate SDE-2 Mastery  
**Scope:** relationship mappings -> ownership -> foreign keys -> cascades -> orphan removal -> fetching -> serialization

## Session Outcome

You should be able to:

- translate an object relationship into a relational schema;
- choose `@ManyToOne`, `@OneToMany`, `@OneToOne`, or `@ManyToMany` deliberately;
- identify the owning side and explain `mappedBy`;
- distinguish a foreign key from a join table;
- keep both sides of a bidirectional relationship consistent in memory;
- choose cascade operations and `orphanRemoval` from aggregate ownership, not habit;
- explain why `LAZY` and `EAGER` are fetch plans rather than business rules;
- prevent JSON recursion and accidental entity-graph exposure;
- design an Order/Customer/Product model suitable for an SDE-2 discussion.

---

## 1. Object Links Are Not Foreign Keys

Java models navigate references. Relational databases store values and constraints.

```text
Java:       order.customer
Database:   orders.customer_id -> customer.id
```

A mapping is a contract between these representations. Every relationship discussion should answer four questions:

1. What is the cardinality?
2. Where is the foreign key stored?
3. Which side writes that foreign key?
4. What lifecycle and fetching behavior is wanted?

A relationship annotation alone does not answer all four. Defaults differ by annotation and provider, and a mapping that compiles can still create an inefficient or unsafe schema.

### Cardinality versus ownership

- **Cardinality** describes how many records can be related.
- **Ownership** describes which mapped association controls the foreign-key or join-table update.
- **Aggregate ownership** describes which object controls lifecycle in the domain.

These are related but not identical. The side that owns a foreign key in JPA is not automatically the domain owner of every lifecycle decision.

---

## 2. `@ManyToOne`: The Usual Foreign-Key Relationship

An order commonly belongs to one customer, while a customer has many orders:

```java
@Entity
public class Order {
    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;
}
```

The database shape is normally:

```text
orders
------
id primary key
customer_id not null foreign key -> customers.id
```

`Order` is normally the owning side because its table contains `customer_id`. The many-to-one association is often the most natural place to start because the foreign key is stored where the many-side row lives.

### Why prefer LAZY for the many-to-one?

A to-one mapping may default differently across JPA annotations, and provider behavior can involve proxies or enhanced classes. Explicitly choosing `LAZY` communicates that loading a customer is not required for every order use case. It does not guarantee zero SQL in every provider scenario; verify the generated SQL and enhancement configuration.

### `optional = false` and database nullability

`optional = false` communicates a required association to JPA. `nullable = false` communicates the database column constraint. For robust design, align both with the domain rule and migration/schema definition.

---

## 3. `@OneToMany`: Inverse Collection and Helper Methods

The customer side can expose its orders:

```java
@Entity
public class Customer {
    @Id
    @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();

    public void addOrder(Order order) {
        orders.add(order);
        order.setCustomer(this);
    }

    public void removeOrder(Order order) {
        orders.remove(order);
        order.setCustomer(null);
    }
}
```

`mappedBy = "customer"` points to the Java field on `Order` that owns the association. It is not a database column name. The customer collection is inverse for foreign-key updates.

### Why helper methods matter

Without a helper method, code can update one side only:

```java
customer.getOrders().add(order);
// order.customer is still null
```

The in-memory graph is inconsistent even if a later persistence operation happens to write the expected foreign key. Domain methods should maintain both sides immediately.

### Collection choice

- `List` preserves order but may need an ordering column if order is persistent.
- `Set` expresses uniqueness but depends on correct `equals()` and `hashCode()` behavior.
- `Collection` avoids promising ordering or uniqueness when neither is a domain rule.

Do not select `Set` merely to hide duplicate-query symptoms. Define identity and equality carefully, especially before an entity receives a generated ID.

---

## 4. Owning Side and `mappedBy`

For a bidirectional association, exactly one side should own the database update.

```java
// owning side
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "customer_id")
private Customer customer;

// inverse side
@OneToMany(mappedBy = "customer")
private List<Order> orders;
```

If code changes only the inverse collection, JPA may not update `orders.customer_id`. The inverse side helps navigation; the owning side controls persistence of the association.

### Interview answer

> The owning side is the association that maps the foreign key or join-table columns and therefore drives relationship updates. `mappedBy` marks the other side as inverse and names the owning Java property; it is not a column name.

### Foreign key location

A many-to-one relation maps naturally to a foreign key on the many table. If the model places the foreign key elsewhere, the mapping may need a join table or a different relationship design. Always draw the schema before writing annotations.

---

## 5. `@OneToOne`

A one-to-one relationship can use a foreign key with a unique constraint:

```java
@Entity
public class User {
    @Id
    private Long id;

    @OneToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "profile_id", nullable = false, unique = true)
    private Profile profile;
}
```

The unique constraint is what prevents multiple users from pointing at the same profile. Application-level intent without a database uniqueness constraint is not enough under concurrency.

### Shared primary key option

A dependent entity can use `@MapsId` when its identity is also the parent's identity:

```java
@Entity
public class UserProfile {
    @Id
    private Long userId;

    @OneToOne
    @MapsId
    @JoinColumn(name = "user_id")
    private User user;
}
```

This expresses strong dependent identity but couples the schema and lifecycle more tightly. Choose it when the dependent cannot exist independently.

### One-to-one caution

To-one lazy loading can be provider-sensitive and may require bytecode enhancement or proxy behavior. If a use case always needs the profile, a DTO query can be clearer than relying on incidental lazy behavior.

---

## 6. `@ManyToMany` and the Association-Entity Alternative

A many-to-many mapping uses a join table:

```java
@Entity
public class Student {
    @Id
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}
```

```text
student_course
--------------
student_id foreign key
course_id  foreign key
unique(student_id, course_id)
```

The join table should normally have a unique constraint across both foreign keys. `CascadeType.REMOVE` is dangerous on a many-to-many association because deleting one student should not delete shared courses.

### Prefer an association entity when the link has data

If enrollment has `enrolledAt`, `role`, or `status`, model it explicitly:

```java
@Entity
public class Enrollment {
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    private Course course;

    private Instant enrolledAt;
    private String status;
}
```

This gives the relationship its own identity, constraints, audit fields, and lifecycle. An explicit entity is usually easier to query and evolve than a bare many-to-many collection.

---

## 7. Cascades: Propagate Operations, Not Ownership Automatically

Cascade controls which `EntityManager` operations propagate from one entity to related entities:

| Cascade | Meaning |
| --- | --- |
| `PERSIST` | persist related new entities |
| `MERGE` | merge related state |
| `REMOVE` | remove related entities |
| `REFRESH` | refresh related state |
| `DETACH` | detach related entities |
| `ALL` | all of the above |

Example:

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL,
           orphanRemoval = true)
private List<OrderLine> lines = new ArrayList<>();
```

This can be sensible when order lines cannot exist independently of an order. It is not automatically sensible for a customer and orders, because deleting a customer may not mean deleting historical orders.

### Cascade is not fetch

Cascade answers “which lifecycle operation propagates?” Fetch answers “when is related data loaded?” `CascadeType.ALL` does not mean EAGER, and `FetchType.EAGER` does not mean cascade persist.

### Cascade is not database `ON DELETE CASCADE`

JPA cascade is an ORM operation behavior. Database cascading is a database constraint/action. They can coexist, but they have different execution, observability, and persistence-context implications.

---

## 8. `orphanRemoval`

`orphanRemoval = true` commonly means that removing a child from the parent's relationship causes the child row to be deleted when the change is synchronized:

```java
order.removeLine(line); // removes from collection and nulls/changes parent
```

It models private ownership: a line has no meaningful life outside its order. It is not the same as `CascadeType.REMOVE`:

- `CascadeType.REMOVE`: deleting the parent propagates remove to children.
- `orphanRemoval`: disassociating a child from the relationship can delete the child.

Using orphan removal on shared or independently reusable entities can cause data loss. Check whether the child may be attached to another aggregate or must be archived rather than physically deleted.

### Collection mutation trap

Replacing a managed collection with a new collection instance can confuse dirty tracking and orphan detection. Prefer domain methods that mutate the managed collection in place and preserve the provider's collection wrapper.

---

## 9. Fetch Type and Entity Graph Shape

`LAZY` and `EAGER` are mapping defaults, not complete query plans.

- **LAZY:** load the association when accessed, if the persistence context and provider can support it.
- **EAGER:** association is required by the mapping, but the provider may load it with a separate select rather than one join.

EAGER does not mean “one efficient query.” It can cause unnecessary work and hidden N+1 behavior. Prefer a default mapping that reflects common usage, then define explicit fetch plans per use case with:

- DTO/projection queries;
- JPQL `join fetch` for suitable queries;
- `@EntityGraph`;
- batch fetching where repeated lazy loads are expected.

The fetch plan belongs close to the use case, not only to the entity annotation.

---

## 10. JSON and API Boundaries

Returning bidirectional entities directly from a REST controller can create:

- infinite recursion: customer -> orders -> customer;
- lazy-loading during serialization;
- oversized responses and accidental PII exposure;
- persistence entities becoming an unstable API contract;
- writes through request-shaped entities that bypass authorization and invariants.

Prefer explicit request/response DTOs:

```java
public record OrderSummaryResponse(
        Long id,
        String status,
        BigDecimal total
) {}
```

Map the exact fields needed by the endpoint. Jackson annotations can stop recursion, but they do not solve query count, authorization, or aggregate design. DTOs and deliberate fetch plans solve the boundary problem more completely.

---

## 11. Design Exercise: Customer, Order, Product

A robust baseline design is:

```text
Customer 1 -------- * Order 1 -------- * OrderLine * -------- 1 Product
```

Recommended choices:

- `Order.customer`: `@ManyToOne(fetch = LAZY)`, owning side, non-null FK.
- `Customer.orders`: inverse `@OneToMany(mappedBy = "customer")`, usually lazy.
- `Order.lines`: `@OneToMany(mappedBy = "order", cascade = {PERSIST, MERGE}, orphanRemoval = true)` if lines are private children.
- `OrderLine.order`: owning many-to-one with non-null FK.
- `OrderLine.product`: many-to-one to a shared product; do not cascade remove product.
- order line stores a snapshot of price/description if historical order meaning must survive product changes.
- API returns DTOs; it does not serialize the entire graph.

### Why not `Order` -> `Product` directly?

The order line is the association entity. It carries quantity, price at purchase, discounts, and possibly tax. A direct many-to-many loses important business data and makes historical behavior harder to model.

### Invariants to enforce

- an order line belongs to exactly one order;
- an order line references one product;
- quantity is positive;
- order total is consistent with lines and discounts;
- deleting a product does not delete historical order lines unless the domain explicitly permits it.

Use database constraints for uniqueness and non-null rules, domain methods for aggregate invariants, and service authorization for cross-aggregate decisions.

---

## 12. Scenario Reasoning

### Scenario A: inverse side only

```java
customer.getOrders().add(order);
orderRepository.save(order);
```

The Java collection changed, but if `order.customer` remains null, the owning side does not know the relationship. Use `customer.addOrder(order)` to update both sides.

### Scenario B: deleting a shared entity

A student is deleted from a many-to-many relationship with courses. `CascadeType.ALL` includes `REMOVE`.

This can attempt to remove courses shared by other students. Many-to-many shared targets normally should not receive remove cascade. Remove the join-table link instead.

### Scenario C: orphan removal

```java
order.getLines().remove(line);
```

With `orphanRemoval = true`, the line can be deleted when the relationship change is synchronized. Without it, the line may remain and require explicit deletion or nullable/disassociation handling. The helper method should also update `line.order`.

### Scenario D: serialization

A controller returns `Order` with lazy `customer` and `lines`.

The serializer may trigger queries, fail after the transaction closes, recurse through bidirectional links, or expose fields not intended for clients. Use a query-specific DTO and fetch exactly its data.

---

## 13. SDE-2 Interview Answers

### What is the owning side?

The owning side is the association that maps the foreign key or join-table columns and controls relationship updates. In a typical customer/order mapping, `Order.customer` owns the `customer_id` foreign key and `Customer.orders` uses `mappedBy = "customer"`.

### What does `mappedBy` mean?

It names the Java property on the other entity that owns the relationship. It prevents a second independent mapping of the same association; it is not a database column name.

### Cascade versus orphan removal?

Cascade propagates selected entity operations from parent to related entities. Orphan removal deletes a privately owned child when it is removed from the parent's relationship. `CascadeType.REMOVE` concerns parent deletion; orphan removal concerns relationship disassociation.

### Why not use `CascadeType.ALL` everywhere?

It can propagate deletes, merges, and other operations across boundaries that are not truly aggregate-owned. That can cause accidental data loss, stale state, and expensive graph traversal. Choose cascades from lifecycle ownership.

### Why is EAGER not a solution to lazy-loading errors?

EAGER can load data unnecessarily, still issue multiple queries, and create performance problems globally. A use case should define an explicit fetch plan and return the data it needs, commonly with a DTO or entity graph.

---

## 14. Practice Batch

1. In a bidirectional `Customer`/`Order` mapping, which side should contain the foreign key and why?
2. What SQL/database structure does a many-to-many mapping require?
3. Why is an association entity often better than many-to-many for a purchase line?
4. Explain `CascadeType.REMOVE` versus `orphanRemoval`.
5. Why can serializing entities directly from a controller create both correctness and performance bugs?

### Model answer key

1. The order table normally contains `customer_id` because each order belongs to one customer. `Order.customer` owns the FK; the customer collection is inverse with `mappedBy`.
2. A join table with both foreign keys and a uniqueness constraint for the pair. The join table is the association owner unless mapped otherwise.
3. A purchase line has quantity, price snapshot, discount, and lifecycle of its own. An explicit entity models those attributes and constraints clearly.
4. Remove cascade propagates deletion of the parent to related entities. Orphan removal deletes a private child when it is removed from the relationship.
5. Serialization can traverse lazy relationships, trigger N+1 queries or fail after detachment, recurse through bidirectional links, and expose internal fields. DTOs and query-specific fetch plans avoid those problems.

---

## Session Handoff

**Prepared study material:** Session 3 — Entity Relationships  
**Topics covered:** cardinality, owning side, `mappedBy`, foreign keys, join tables, all major relationship annotations, cascades, orphan removal, fetch type, bidirectional consistency, JSON boundaries, and Order/Customer/Product design.  
**Next session:** Session 4 — Transactions and Spring Transaction Management  
**Next starting topic:** `@Transactional` -> Spring proxy/interceptor -> transaction manager -> persistence-context boundary

## One-Line Revision Summary

Map the database first: the owning side writes the FK or join table, cascades propagate selected operations, orphan removal expresses private child ownership, and fetch/serialization choices must be designed per use case rather than left to defaults.
