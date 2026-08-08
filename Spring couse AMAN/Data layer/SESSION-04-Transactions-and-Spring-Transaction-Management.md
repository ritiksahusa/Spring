# Session 4 — Transactions and Spring Transaction Management

**Track:** JPA / Hibernate SDE-2 Mastery  
**Scope:** `@Transactional` -> Spring proxy -> transaction manager -> persistence context -> flush/commit/rollback -> propagation/isolation -> production boundaries

## Session Outcome

You should be able to:

- trace a Spring service call through the transaction interceptor;
- distinguish a transaction from a persistence context and a database connection;
- explain when an EntityManager is joined to a transaction;
- reason about flush, commit, rollback, and exception timing;
- choose propagation behavior for common service workflows;
- distinguish isolation from propagation;
- explain read-only transactions without overclaiming what they enforce;
- identify self-invocation, private/final method, and proxy-boundary pitfalls;
- reason about checked versus unchecked rollback rules;
- explain transaction synchronization and OSIV trade-offs.

---

## 1. Three Boundaries to Keep Separate

A reliable transaction explanation separates:

1. **Spring transaction boundary** — the proxy starts, joins, suspends, or ends a transaction.
2. **Persistence-context boundary** — managed entities and dirty checking live in an EntityManager context.
3. **Database transaction/connection boundary** — the database makes isolation and durability decisions.

They often line up in a standard Spring Boot service, but they are not synonyms.

```text
client request
    -> Spring proxy/interceptor
    -> transaction manager obtains or joins transaction
    -> EntityManager joins persistence context
    -> service reads/mutates managed entities
    -> flush synchronizes ORM state
    -> database commit or rollback
```

A repository method is not automatically the business transaction boundary. Put the boundary around a meaningful use case, usually at the service layer.

---

## 2. What `@Transactional` Does

```java
@Service
public class PaymentService {
    @Transactional
    public void capture(Long paymentId) {
        Payment payment = payments.findById(paymentId).orElseThrow();
        payment.capture();
        ledger.record(payment);
    }
}
```

Conceptually, Spring creates a proxy around the service. On an external call to `capture()` it:

1. reads transaction metadata;
2. asks the transaction manager to create or join a transaction;
3. invokes the target method;
4. commits if the method succeeds according to the configured rules;
5. rolls back if the rules classify the outcome for rollback;
6. cleans up resources and synchronizations.

The actual transaction manager may coordinate JPA, JDBC, and other resources. With JPA, it coordinates the EntityManager and database transaction.

### The annotation does not make arbitrary code transactional

If code runs outside the proxy, or the method is not eligible for interception, the annotation may not take effect. Also, `@Transactional` does not make remote calls, external queues, or multiple databases atomically consistent by itself.

---

## 3. EntityManager and Transaction Boundary

Within a typical transaction:

```java
@Transactional
public void rename(Long id, String name) {
    Customer customer = entityManager.find(Customer.class, id);
    customer.setName(name); // managed; dirty checking
} // flush/commit path
```

The entity becomes managed in the persistence context associated with the unit of work. At commit, the provider generally flushes pending changes before the database commit.

A repository can also be called without a surrounding service transaction for simple reads, depending on configuration. The important question is not “does this repository method have an annotation?” but:

- which transaction is active at the call;
- which persistence context is active;
- when does it end;
- can lazy access and dirty checking occur safely there?

### Open versus joined persistence context

An EntityManager may exist without an active database transaction, especially for read operations. A write needs a transaction. A managed entity outside the intended transaction can create confusing behavior: lazy access may fail, changes may not flush, or a later merge may be required.

---

## 4. Flush, Commit, and Rollback

```text
method changes managed entity
        |
        v
flush: ORM sends INSERT/UPDATE/DELETE to database
        |
        v
commit: database makes transaction durable
```

A flush is not a commit. SQL can be sent successfully and the transaction can still roll back later.

Rollback can happen because of:

- an unchecked exception classified for rollback;
- a checked exception with explicit rollback configuration;
- a constraint violation discovered during flush/commit;
- a timeout or connection/database failure;
- application code marking the transaction rollback-only.

### Flush can expose errors late

```java
@Transactional
public void create(Invoice invoice) {
    invoices.save(invoice);
    // A constraint violation may appear here, at flush, or during commit.
}
```

The repository call is not always the moment the database rejects the write. If a method catches an exception after the transaction has already been marked rollback-only, returning normally may still result in an `UnexpectedRollbackException` at the outer boundary.

### Explicit flush

Use `flush()` when the current use case needs synchronization before continuing. Do not use it as a synonym for commit or as a blanket fix for transaction bugs.

---

## 5. Propagation

Propagation answers: **What should this method do when a transaction already exists?**

| Propagation | Existing transaction | No existing transaction | Typical use |
| --- | --- | --- | --- |
| `REQUIRED` | Join it | Create one | Default service use case |
| `REQUIRES_NEW` | Suspend it; create new one | Create one | Independent audit/outbox work with separate outcome |
| `SUPPORTS` | Join it | Run without one | Optional transaction context |
| `MANDATORY` | Join it | Fail | Must be called inside transaction |
| `NOT_SUPPORTED` | Suspend it | Run without one | Explicitly non-transactional work |
| `NEVER` | Fail if one exists | Run without one | Rare strict boundary |
| `NESTED` | Savepoint where supported | Create one | Partial rollback within one physical transaction, provider/manager dependent |

### `REQUIRED`

```java
@Transactional
public void placeOrder(...) {
    reserveInventory(); // joins same transaction if REQUIRED
    chargePayment();    // joins same transaction if local transaction manager
}
```

A failure in a joined method can mark the whole transaction rollback-only.

### `REQUIRES_NEW`

It suspends the outer transaction and starts an independent one. This can be correct for an audit record that must commit even when the main operation fails, but it can also:

- require another database connection;
- increase pool pressure;
- create surprising visibility/order behavior;
- break assumptions about atomicity.

Do not use it merely to silence rollback errors.

### `NESTED`

Nested behavior usually depends on savepoint support and transaction manager configuration. It is not the same as `REQUIRES_NEW`: nested work shares the physical transaction and can roll back to a savepoint, while `REQUIRES_NEW` uses a separate transaction.

---

## 6. Isolation

Isolation answers: **What can concurrent transactions observe and how are their reads/writes ordered?**

Common levels:

| Level | Main guarantee/trade-off |
| --- | --- |
| `READ_UNCOMMITTED` | permits dirty reads on databases that support it |
| `READ_COMMITTED` | avoids dirty reads; non-repeatable reads may occur |
| `REPEATABLE_READ` | stronger repeat-read behavior; implementation varies |
| `SERIALIZABLE` | strongest isolation; lower concurrency and possible contention |

Isolation is not propagation. `REQUIRED` can join a transaction whose isolation was configured separately. A transaction's isolation also does not automatically prevent application-level lost updates; use versioning or locks where needed.

### Practical reasoning

When a user submits an update based on an earlier read, ask:

- Can another transaction update the row first?
- Is there a version column?
- Is the update conditional on the version/current value?
- Should the operation lock the row?
- What should the API return on conflict?

Do not solve every concurrency problem by choosing `SERIALIZABLE`; it can reduce throughput and produce deadlocks or timeouts.

---

## 7. Rollback Rules

By default, Spring commonly rolls back for unchecked exceptions and errors, while checked exceptions may require explicit configuration:

```java
@Transactional(rollbackFor = ExternalPaymentException.class)
public void settle(...) throws ExternalPaymentException {
    // checked exception should also mark this transaction for rollback
}
```

The exact behavior depends on Spring configuration and the exception crossing the transactional boundary. A caught exception does not automatically undo work if it is swallowed:

```java
@Transactional
public void process() {
    try {
        repository.save(entity);
        externalCall();
    } catch (ExternalException ex) {
        log.warn("failed", ex);
        // method returns normally; transaction may commit unless marked rollback-only
    }
}
```

If the business rule requires rollback, rethrow an appropriate exception or mark the transaction rollback-only deliberately. Do not rely on log messages to express transaction intent.

---

## 8. Self-Invocation and Proxy Pitfalls

This is a classic interview scenario:

```java
@Service
public class ImportService {
    public void importAll(List<Row> rows) {
        rows.forEach(row -> saveOne(row)); // direct call on this
    }

    @Transactional
    public void saveOne(Row row) {
        repository.save(row);
    }
}
```

If `importAll()` is called through the proxy, its direct `this.saveOne()` calls bypass the proxy. The `@Transactional` interceptor on `saveOne()` may not run.

Safer designs:

- put the transactional method on another injected service;
- make the outer method the transaction boundary when one transaction is intended;
- use a self-injected proxy only with clear justification;
- avoid relying on private/final methods that cannot be intercepted by the chosen proxy mechanism.

Other proxy traps:

- calls from constructors occur before proxying is complete;
- direct calls on the target object bypass advice;
- final classes/methods may limit subclass proxies;
- package/proxy configuration can alter what is intercepted.

The diagnostic question is always: **Did the call cross the Spring proxy?**

---

## 9. Read-Only Transactions

```java
@Transactional(readOnly = true)
public OrderView getOrder(Long id) { ... }
```

Read-only is a performance and intent hint, not a universal write firewall. Depending on transaction manager and provider, it can influence flush mode or connection settings. It does not mean:

- the database cannot physically accept a write;
- all accidental mutations are impossible;
- every provider behaves identically;
- lazy associations are automatically safe after return.

Use read-only for clearly read-oriented use cases and still avoid mutating managed entities in them.

---

## 10. Transaction Synchronization and OSIV

Transaction synchronization lets resources run callbacks around transaction completion, for example:

- before commit;
- after commit;
- after rollback;
- after completion.

This is useful for publishing an event after durable commit rather than publishing before a rollback. For reliable cross-system delivery, an outbox pattern is usually stronger than a callback that can be lost after process failure.

### OSIV: Open Session in View

OSIV keeps a persistence context available through the web view/request after the service transaction. It can prevent some lazy-loading exceptions in controllers, but it can also:

- allow SQL during serialization;
- hide N+1 queries;
- hold connections or persistence state longer than expected depending on configuration;
- blur the service transaction boundary;
- make API latency depend on view traversal.

A deliberate service-layer fetch plan and DTO mapping usually make data access easier to reason about. If OSIV is enabled, treat it as an explicit operational trade-off, not proof that transaction boundaries are correct.

---

## 11. Service Boundary Design

A good transaction boundary usually surrounds one business use case:

```java
@Transactional
public void approveLoan(LoanApprovalCommand command) {
    Loan loan = loans.findById(command.loanId()).orElseThrow();
    borrowerPolicy.check(loan, command);
    loan.approve();
    approvalRepository.save(new ApprovalRecord(loan));
}
```

The transaction should include the database state changes that must succeed or fail together. Avoid:

- holding a transaction open across slow remote calls unless the trade-off is explicit;
- doing user interaction inside a transaction;
- mixing unrelated batch items in one giant transaction without memory/error planning;
- assuming multiple databases or message brokers participate atomically.

For remote side effects, consider outbox, idempotency, compensation, or a workflow rather than pretending `@Transactional` spans the network.

---

## 12. Scenario Reasoning

### Scenario A: caught exception

A method saves an entity, catches a checked exception, logs it, and returns.

**Answer:** The transaction may commit because the exception was caught and may not match rollback rules. Decide explicitly whether to rethrow or mark rollback-only.

### Scenario B: self-invocation

A non-transactional method calls an annotated method on `this`.

**Answer:** The call can bypass Spring's proxy, so the annotation may not be applied. Move the boundary or call through a proxied bean.

### Scenario C: SQL succeeded, method failed later

An `UPDATE` appears in logs, then a later operation throws.

**Answer:** The SQL may have been flushed but the transaction can still roll back. Inspect the final transaction outcome, not only the SQL log.

### Scenario D: `REQUIRES_NEW` audit

An outer transaction updates an order, then an inner `REQUIRES_NEW` audit commits, and the outer transaction rolls back.

**Answer:** The audit can remain while the order update disappears. That is correct only if independent audit durability is intended. It also consumes another connection while the outer one is suspended.

### Scenario E: OSIV masking design

An endpoint returns entities and serialization triggers 200 lazy queries.

**Answer:** OSIV kept the context available but did not create an efficient fetch plan. Measure SQL, define a DTO query/entity graph/batch strategy, and keep the service boundary intentional.

---

## 13. SDE-2 Interview Answers

### What does `@Transactional` do?

Spring applies transaction advice through a proxy. On an eligible external call it creates or joins a transaction, invokes the method, and commits or rolls back according to outcome and configuration. With JPA, the EntityManager and persistence context participate in that unit of work. The annotation does not make self-invocation or arbitrary remote work transactional.

### Transaction versus persistence context?

A transaction is an atomicity/isolation/durability boundary, primarily at the database/resource level. A persistence context is the set of managed entity instances and tracking state. Spring commonly aligns them around a service call, but they are separate concepts and can have different lifetimes.

### Why is flush not commit?

Flush sends ORM changes as SQL to the database. Commit makes the transaction durable. SQL can be visible to the current transaction and still be rolled back later.

### `REQUIRED` versus `REQUIRES_NEW`?

`REQUIRED` joins an existing transaction or creates one. `REQUIRES_NEW` suspends the existing transaction and starts an independent one. The latter can commit independently but uses another connection and changes atomicity semantics.

### Why can self-invocation break transactions?

The direct call does not cross the Spring proxy, so the transactional interceptor is bypassed. The method runs with whatever transaction context already exists, not necessarily the one declared on that method.

---

## 14. Practice Batch

1. Explain the call path from an external service call to database commit.
2. Why can a checked exception fail to roll back by default?
3. Why is `REQUIRES_NEW` risky under connection-pool pressure?
4. What does `readOnly = true` promise, and what does it not promise?
5. Why can OSIV hide a bad fetch plan?

### Model answer key

1. The call crosses a Spring proxy, transaction advice creates or joins a transaction, the EntityManager/persistence context tracks work, flush synchronizes SQL, and the transaction manager commits or rolls back.
2. Rollback rules commonly classify unchecked exceptions for rollback while checked exceptions need explicit configuration. The rule can be customized.
3. It suspends the outer transaction while holding its connection and obtains another connection for the inner transaction. Under pool pressure this can block or exhaust the pool.
4. It communicates read intent and may influence flush/connection behavior. It is not a universal prohibition on writes and is provider/configuration dependent.
5. OSIV keeps the context available during serialization, so lazy accesses succeed while silently issuing extra SQL. The endpoint can still be slow and violate service-layer fetch expectations.

---

## Session Handoff

**Prepared study material:** Session 4 — Transactions and Spring Transaction Management  
**Topics covered:** transaction and persistence-context boundaries, `@Transactional`, proxy/interceptor flow, flush/commit/rollback, propagation, isolation, rollback rules, read-only transactions, self-invocation, synchronization, OSIV, and service design.  
**Next session:** Session 5 — Fetching and N+1  
**Next starting topic:** lazy loading -> `LazyInitializationException` -> identifying an N+1 query from SQL evidence

## One-Line Revision Summary

`@Transactional` is proxy-driven orchestration around a resource transaction; persistence-context tracking, flush timing, rollback rules, propagation, isolation, and fetch behavior must be reasoned about separately even when they meet in one service method.
