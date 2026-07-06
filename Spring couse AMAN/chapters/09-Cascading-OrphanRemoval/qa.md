# Chapter 09 — QA: Cascading, OrphanRemoval and Mapping Operations

---

## Part A — Short Answer Questions (20)

**A1.** What is cascading in JPA? Give a one-line definition.

> **Answer:** Cascading means: when an operation (save, update, delete, etc.) is performed on the parent entity, the same operation is automatically propagated to the associated child entity.

---

**A2.** What is the difference between CascadeType.PERSIST and CascadeType.MERGE?

> **Answer:**
> - `PERSIST` triggers when a parent entity is saved for the FIRST TIME (`entityManager.persist`). If the child is new (transient), it also inserts the child.
> - `MERGE` triggers when an existing parent entity is updated. If the child has changes or is new, it merges/inserts the child too.
> In Spring Data JPA, `repository.save()` uses either persist (new) or merge (existing), so having both is typical.

---

**A3.** What error occurs when you try to save a parent entity that references a new (unsaved) child, without cascade?

> **Answer:** `org.hibernate.TransientPropertyValueException: object references an unsaved transient instance — save the transient instance before flushing`. JPA requires all referenced objects to be in persistent state, unless cascaded.

---

**A4.** What is `CascadeType.REMOVE`? When is it triggered?

> **Answer:** `CascadeType.REMOVE` causes all child entities to be deleted when the parent entity is deleted. It is triggered when `repository.delete(parent)` or `entityManager.remove(parent)` is called.

---

**A5.** What is `orphanRemoval = true`? How is it different from CascadeType.REMOVE?

> **Answer:**
> - `orphanRemoval = true` deletes a child entity when it is **removed from the parent's collection** OR the parent is deleted — even when the parent itself is NOT deleted.
> - `CascadeType.REMOVE` only deletes children when the parent entity is explicitly deleted.
> Example: `patient.setInsurance(null)` with `orphanRemoval=true` deletes the old insurance from DB. Without `orphanRemoval`, it would just nullify the FK.

---

**A6.** Can you put `orphanRemoval = true` on a `@ManyToMany` mapping?

> **Answer:** No. `orphanRemoval = true` is only valid on `@OneToOne` and `@OneToMany`. It is not allowed on `@ManyToMany` because a many-to-many child can have multiple parents, so it cannot be an "orphan" in the same sense.

---

**A7.** What is the difference between the JPA domain concept of "Owning Side" and the Data domain concept of "Parent"?

> **Answer:**
> - **JPA domain** (Owning/Inverse): Determined by `@JoinColumn` placement. The owning side manages FK updates. Example: `Appointment` has `@ManyToOne` with `@JoinColumn` → Appointment is owning side.
> - **Data domain** (Parent/Child): Determined by which entity can exist independently. `Patient` can exist without an appointment → Patient is the Parent. Appointment cannot exist without Patient → Appointment is the Child.
> These are separate concepts. The owning side in JPA can be the child in the data domain.

---

**A8.** Why is it dangerous to put `CascadeType.REMOVE` on a `@ManyToOne` (child → parent direction)?

> **Answer:** Deleting one appointment would try to cascade-delete its patient. The patient may have other appointments, causing FK constraint violations or unintentionally wiping out data. Cascade should only go from parent to child, not from child to parent.

---

**A9.** If you are inside a `@Transactional` method and you modify a persistent entity without calling `.save()`, will the changes be persisted?

> **Answer:** Yes. Inside a `@Transactional` method, entities loaded from DB are in the **persistent state** and tracked by the PersistenceContext. When the transaction commits, Hibernate performs **dirty checking** — it compares current state to the snapshot taken at load time, and auto-issues `UPDATE` SQL for any changes.

---

**A10.** Why do you need `@Transactional` on the `assignInsuranceToPatient` service method?

> **Answer:** Without `@Transactional`, the PersistenceContext closes after each repository call. To keep the patient in persistent state and benefit from dirty checking (auto-save on method return), the entire method must run within one transaction. `@Transactional` opens a transaction at method entry and commits it when the method returns.

---

**A11.** When creating a new `Appointment` entity and saving it, why do you need to explicitly call `appointmentRepository.save(appointment)`?

> **Answer:** A newly created appointment is in **transient state** — it has no ID and is not tracked by the PersistenceContext. Dirty checking only works on entities already in persistent state (loaded from DB). New (transient) entities must be explicitly saved to enter the persistent state.

---

**A12.** Why do you NOT need to call `.save()` when reassigning an appointment's doctor?

> **Answer:** The appointment was loaded from the DB with `findById()` → it is in **persistent state**. Setting `appointment.setDoctor(newDoctor)` makes it dirty. When the `@Transactional` method returns and the transaction commits, dirty checking auto-issues the UPDATE SQL.

---

**A13.** What is bidirectional consistency and why is it important?

> **Answer:** When two entities have a bidirectional mapping, both the owning side and the inverse side Java objects must reflect the same state. Example: when adding an appointment to a patient, you should both call `appointment.setPatient(patient)` (owning side — needed for DB) AND `patient.getAppointments().add(appointment)` (inverse side — for in-memory accuracy). Without the inverse-side update, the DB is correct, but accessing `patient.getAppointments()` in the same session would return a stale list.

---

**A14.** What is the default FetchType for `@OneToMany`?

> **Answer:** `FetchType.LAZY`. The collection is not loaded from DB when the parent is fetched. It loads only when you explicitly access it (e.g., `patient.getAppointments()`), and only if a session/transaction is still open.

---

**A15.** What is the default FetchType for `@ManyToOne` and `@OneToOne`?

> **Answer:** `FetchType.EAGER`. The related entity is loaded automatically whenever the parent is fetched, in the same query.

---

**A16.** What is `LazyInitializationException` and when does it occur?

> **Answer:** `LazyInitializationException` occurs when you try to access a lazy-loaded collection or relationship **after the Hibernate session/PersistenceContext has been closed**. The session closes at the end of a `@Transactional` block (or after each repository call without a transaction). Common message: `"could not initialize proxy — no Session"`.

---

**A17.** List two ways to fix `LazyInitializationException`.

> **Answer:**
> 1. Add `@Transactional` to the method that accesses the lazy relationship — keeps the session open until the method returns.
> 2. Add `@ToString.Exclude` on the lazy field — prevents the toString (triggered by logging/printing) from accessing the lazy relationship.
> (Also: change to `FetchType.EAGER` — but avoid for collections as it causes N+1 problems.)

---

**A18.** What does `@Builder` from Lombok do?

> **Answer:** `@Builder` generates a static `builder()` method on the class, which returns a builder object with fluent setter methods for each field. You call `.build()` to create the final object. It avoids the need to remember constructor argument order.

---

**A19.** When should cascade be defined on the parent side vs the child side?

> **Answer:** Cascade should always be defined on the **parent side** — the entity that dictates the lifecycle of the child. For example, Patient (parent) → Appointment (child): define cascade on the `@OneToMany appointments` field in Patient. Defining cascade on the child's `@ManyToOne` to parent is dangerous as it can accidentally delete the parent when the child is deleted.

---

**A20.** Write the annotation to cascade only persist and merge from Patient to Insurance (OneToOne).

> **Answer:**
> ```java
> @OneToOne(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
> @JoinColumn(name = "patient_insurance_id")
> private Insurance insurance;
> ```

---

## Part B — Multiple Choice Questions (12)

**B1.** Which CascadeType is triggered when `repository.save()` is called on a NEW parent entity?

- a) MERGE
- b) PERSIST ✅
- c) REMOVE
- d) REFRESH

---

**B2.** `orphanRemoval = true` deletes the child when:

- a) The parent entity is deleted
- b) The child's reference is set to null or removed from collection ✅ (also when parent deleted)
- c) The child's own delete method is called
- d) The transaction rolls back

---

**B3.** What is the default FetchType for `@OneToMany`?

- a) EAGER
- b) LAZY ✅
- c) DEFERRED
- d) AUTO

---

**B4.** Without `@Transactional`, which of the following happens?

- a) Dirty checking still works because JPA always checks
- b) Entities remain in persistent state indefinitely
- c) Session closes after each repository call; persistent state is lost ✅
- d) All save operations are auto-committed

---

**B5.** `CascadeType.REMOVE` vs `orphanRemoval=true`: which one deletes children only when the parent itself is deleted?

- a) `orphanRemoval = true`
- b) `CascadeType.REMOVE` ✅
- c) Both behave identically
- d) Neither

---

**B6.** Which entity has the FK column in a `@ManyToOne` relationship?

- a) The parent entity
- b) The entity with `@ManyToOne` (child/owning side) ✅
- c) Both entities get a FK column
- d) A join table is created

---

**B7.** A new (transient) entity must be saved explicitly because:

- a) JPA doesn't support dirty checking
- b) Transient entities are not tracked by PersistenceContext and dirty checking only works on tracked entities ✅
- c) Spring Boot security blocks unsaved entities
- d) Only EAGER entities are auto-saved

---

**B8.** Which annotation combination is NOT valid?

- a) `@OneToMany(orphanRemoval = true)`
- b) `@OneToOne(orphanRemoval = true)`
- c) `@ManyToMany(orphanRemoval = true)` ✅ — not supported
- d) `@OneToMany(cascade = CascadeType.ALL)`

---

**B9.** In the reassign-doctor operation, why doesn't Hibernate re-use the previously loaded doctor object from the prior transaction?

- a) Hibernate never caches doctor objects
- b) The new method is in a different transaction — the old PersistenceContext is closed ✅
- c) Doctor is marked as final and cannot be reused
- d) `@Transactional` clears all caches on entry

---

**B10.** What does `patient.getAppointments().add(appointment)` achieve when called in the same transaction?

- a) It saves the appointment to the database
- b) It updates the appointment's patient FK immediately
- c) It maintains bidirectional consistency in the in-memory object graph ✅
- d) It triggers an INSERT SQL

---

**B11.** You call `patient.setInsurance(null)` inside a `@Transactional` method. With `orphanRemoval = true`, what SQL is generated?

- a) `UPDATE patient SET insurance_id = NULL`
- b) `UPDATE patient SET insurance_id = NULL; DELETE FROM insurance WHERE id = ?` ✅
- c) `DELETE FROM patient WHERE id = ?`
- d) Nothing — orphanRemoval only works on collections

---

**B12.** `CascadeType.ALL` includes:

- a) PERSIST, MERGE, REMOVE
- b) PERSIST, MERGE, REMOVE, REFRESH, DETACH ✅
- c) PERSIST, REMOVE only
- d) MERGE, REMOVE, DETACH only

---

## Part C — Scenario and Code Questions (10)

**C1.** You have Patient → Insurance (OneToOne). You want saving a patient to auto-save a new insurance. Write the correct annotation.

> **Answer:**
> ```java
> // Patient.java
> @OneToOne(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
> @JoinColumn(name = "patient_insurance_id")
> private Insurance insurance;
> ```

---

**C2.** Write a service method that deletes a patient and automatically deletes all their appointments.

> **Answer:**
> ```java
> // Patient.java — define cascade
> @OneToMany(mappedBy = "patient", cascade = CascadeType.REMOVE)
> private List<Appointment> appointments = new ArrayList<>();
>
> // AppointmentService.java
> @Transactional
> public void deletePatient(Long patientId) {
>     Patient patient = patientRepository.findById(patientId).orElseThrow(...);
>     patientRepository.delete(patient);
>     // Hibernate: DELETE FROM appointment WHERE patient_id = ?
>     //            DELETE FROM patient WHERE id = ?
> }
> ```

---

**C3.** Explain what happens step by step when this code runs:

> ```java
> @Transactional
> public Patient dissociateInsurance(Long patientId) {
>     Patient patient = patientRepository.findById(patientId).orElseThrow();
>     patient.setInsurance(null);
>     return patient;
> }
> ```
> Assume Patient has `@OneToOne(orphanRemoval = true)` on the insurance field.

> **Answer:**
> 1. `findById` → SELECT patient from DB → patient is now in persistent state
> 2. `patient.setInsurance(null)` → patient is marked as dirty; old insurance reference is removed → insurance is now "orphaned"
> 3. Method returns → transaction commits
> 4. Dirty checking detects patient changed → issues `UPDATE patient SET insurance_id = NULL`
> 5. orphanRemoval detects old insurance has no parent → issues `DELETE FROM insurance WHERE id = <old_id>`

---

**C4.** A developer writes this code in a non-@Transactional method. What error will occur and why?

> ```java
> Patient patient = patientRepository.findById(1L).orElseThrow();
> System.out.println(patient.getAppointments().size());
> ```

> **Answer:** `LazyInitializationException: could not initialize proxy - no Session`. Without `@Transactional`, the Hibernate session closes after `findById`. `appointments` is lazy (`@OneToMany` default). Accessing it after the session closes fails.

---

**C5.** Identify the bug in this code:

> ```java
> @ManyToOne(cascade = CascadeType.REMOVE)
> @JoinColumn(name = "patient_id", nullable = false)
> private Patient patient;
> ```

> **Answer:** `CascadeType.REMOVE` is placed on a `@ManyToOne` child-to-parent relationship. This means: when you delete any appointment, Hibernate will also try to delete the patient. If that patient has other appointments, you'll get a FK constraint violation. Even if the patient has no other appointments, the patient data is permanently deleted — which is almost never desired. Cascade should go from parent to child, never child to parent.

---

**C6.** Write the builder-pattern creation of an Appointment with reason "Annual checkup" and date 2025-03-15 at 10:00.

> **Answer:**
> ```java
> Appointment appointment = Appointment.builder()
>     .reason("Annual checkup")
>     .appointmentTime(LocalDateTime.of(2025, 3, 15, 10, 0, 0))
>     .build();
> ```
> Note: `@Builder @NoArgsConstructor @AllArgsConstructor` must be on the Appointment entity.

---

**C7.** What happens when you call `patient.getAppointments().remove(appt)` inside a `@Transactional` method, given `@OneToMany(orphanRemoval = true)`?

> **Answer:** When the transaction commits, Hibernate:
> 1. Detects that `appt` was removed from the patient's appointments collection
> 2. Since `orphanRemoval = true`, it issues `DELETE FROM appointment WHERE id = <appt.id>`
> 3. The patient itself is NOT deleted — only the orphaned appointment

---

**C8.** You have this annotation on Patient.insurance: `@OneToOne(cascade = {CascadeType.PERSIST, CascadeType.MERGE})`. A test does:
> ```java
> Insurance ins = Insurance.builder().policyNumber("X1").provider("LIC").validUntil(LocalDate.now()).build();
> insuranceService.assignInsuranceToPatient(ins, 1L);
> ```
> Is a separate `insuranceRepository.save(ins)` call needed?

> **Answer:** No. Inside `assignInsuranceToPatient`, when `patient.setInsurance(ins)` is called and the transaction commits:
> - `CascadeType.PERSIST` causes Hibernate to also insert the insurance
> - The patient's `insurance_id` FK is then updated automatically
> No explicit save of insurance is needed.

---

**C9.** Compare the SQL generated for these two scenarios when the transaction ends:

> **Scenario A:** `appointment.setDoctor(newDoctor)` inside `@Transactional`
> **Scenario B:** `appointmentRepository.save(appointment)` without `@Transactional`

> **Answer:**
> - **Scenario A:** `UPDATE appointment SET doctor_id = <newDoctor.id> WHERE id = <appointment.id>` — generated by dirty checking at transaction commit. No explicit save call needed.
> - **Scenario B:** Without `@Transactional`, there is no persistent state tracking. `save()` calls `merge()` on the entity, and Spring Data JPA issues `UPDATE appointment SET ... WHERE id = ?` immediately.
> Both produce the same DB result but via different mechanisms. Scenario A relies on dirty checking + `@Transactional`; Scenario B is explicit.

---

**C10.** What is the difference between Parent-Child (Data Domain) and Owning-Inverse (JPA Domain) for the Appointment entity?

> **Answer:**
> - **Owning Side (JPA):** Appointment has `@ManyToOne` with `@JoinColumn` → Appointment is the owning side for both patient and doctor relationships. It controls the FK columns.
> - **Child (Data Domain):** Appointment cannot exist without a Patient or Doctor → Appointment is the child. Patient and Doctor are parents.
> These concepts are independent. The owning side (Appointment) in JPA can be the child in the data domain.

---

## Part D — Fill in the Blanks (8)

**D1.** Cascading makes child entities perform the ______ operation as the parent.

> **Answer:** same (propagated)

---

**D2.** `CascadeType.PERSIST` is triggered when `repository.save()` is called on a ______ entity.

> **Answer:** new (transient)

---

**D3.** `orphanRemoval = true` removes a child entity when it is ______ from the parent's collection or its reference is set to ______.

> **Answer:** removed / null

---

**D4.** The ______ checking feature of Hibernate auto-issues UPDATE SQL when a persistent entity is modified inside a `@Transactional` method.

> **Answer:** dirty

---

**D5.** Default FetchType for `@OneToMany` is ______, meaning the collection is not loaded until explicitly accessed.

> **Answer:** LAZY

---

**D6.** A newly created entity is in ______ state, and must be explicitly saved before Hibernate can track it.

> **Answer:** transient

---

**D7.** `CascadeType.REMOVE` deletes children ______ when the parent is deleted, while `orphanRemoval = true` deletes children whenever they are ______ from the parent.

> **Answer:** only / removed (disassociated)

---

**D8.** `@Builder` from Lombok generates a static ______ method that returns a fluent builder for object creation.

> **Answer:** `builder()`

---

## Part E — True or False (8)

**E1.** `CascadeType.ALL` includes both PERSIST and REMOVE.

> **Answer:** True — CascadeType.ALL includes PERSIST, MERGE, REMOVE, REFRESH, and DETACH.

---

**E2.** Dirty checking works on transient entities (entities that have never been saved).

> **Answer:** False — Dirty checking only works on entities in the **persistent state** (loaded from DB and tracked by PersistenceContext). Transient entities are not tracked.

---

**E3.** `orphanRemoval = true` can be used on `@ManyToMany` relationships.

> **Answer:** False — orphanRemoval is only valid on `@OneToOne` and `@OneToMany`.

---

**E4.** Without `@Transactional`, calling `patient.setInsurance(insurance)` and returning the patient will auto-save the insurance to the DB.

> **Answer:** False — without `@Transactional`, the PersistenceContext is closed and dirty checking doesn't run. No UPDATE/INSERT will happen just from setting a field.

---

**E5.** It is safe to put `CascadeType.REMOVE` on a `@ManyToOne` in a child entity.

> **Answer:** False — This would delete the parent when the child is deleted. A parent typically has many children, so this could trigger cascaded deletes of all sibling children and break the application.

---

**E6.** `FetchType.EAGER` on `@OneToMany` loads the entire collection every time the parent is fetched.

> **Answer:** True — but this is discouraged for OneToMany because it can return duplicate parent rows (Hibernate joins the collection) and cause performance issues (N+1 or excessive data).

---

**E7.** Updating the inverse side of a bidirectional relationship inside a `@Transactional` method updates the FK in the database.

> **Answer:** False — only the owning side controls FK updates. Updating only the inverse side collection does not change the FK in DB. You must update the owning side.

---

**E8.** When `CascadeType.PERSIST` is set on a `@OneToOne`, Hibernate will insert the child entity if it is in transient state when the parent is saved.

> **Answer:** True — CascadeType.PERSIST propagates the persist (insert) operation from parent to child. The child is inserted first, its generated ID is then stored as FK in the parent.

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

**F1.** You have `cascade = CascadeType.ALL` AND `orphanRemoval = true` on the SAME `@OneToMany`. Is this redundant?

> **Answer:** Not fully. `CascadeType.ALL` (its `REMOVE`) handles "parent deleted → delete all children." `orphanRemoval = true` handles a DIFFERENT case: "child removed from the parent's collection while the parent STILL EXISTS" (e.g., `patient.getAppointments().remove(appt)`) — `CascadeType.REMOVE` alone would NOT delete that orphan. Using both together is the correct pattern for "child cannot exist outside the parent's collection."

**F2.** What is the "helper method" pattern for bidirectional relationships with cascade + orphanRemoval, and what bug does it prevent?

> **Answer:** Instead of callers manually setting BOTH sides separately (easy to forget one), add symmetric methods on the parent: `addAppointment()` (adds to the collection AND sets the child's back-reference) and `removeAppointment()` (removes from the collection AND nulls the back-reference). This guarantees both sides of the relationship stay in sync, preventing the classic bug where only one side is updated — either the in-memory graph is stale, or the DB FK never changes.

**F3.** Why can removing an item from a Hibernate-managed `List` inside a `for-each` loop throw `ConcurrentModificationException`, and how does `orphanRemoval` make this easier to trigger accidentally?

> **Answer:** Removing from a collection mid-iteration with a standard `for-each` (using an `Iterator` internally) throws `ConcurrentModificationException` because the modification count changes during iteration — a plain Java collections issue, but especially easy to hit when writing orphanRemoval-triggering cleanup logic (e.g., "remove all appointments older than X"). Fix: use `Iterator.remove()`, `collection.removeIf(predicate)`, or iterate over a defensive copy.

**F4.** A `@Transactional` method partially updates several entities, then throws near the end. Does `orphanRemoval`'s DELETE get rolled back too?

> **Answer:** Yes — the ENTIRE transaction rolls back, including any SQL Hibernate already flushed earlier in the SAME transaction (e.g., an auto-flush before a mid-method query). Rollback operates at the database transaction level, not at the ORM/dirty-checking level, so cascaded/orphan-triggered DELETEs are undone along with everything else.

**F5.** Why is `cascade = CascadeType.ALL` dangerous on a `@ManyToMany` like `Department.doctors`?

> **Answer:** `CascadeType.REMOVE` (included in `ALL`) would attempt to cascade-delete every `Doctor` in a deleted department's set — but doctors are typically SHARED across multiple departments in a many-to-many. This could delete a doctor still needed by OTHER departments, corrupting data well beyond the department being deleted. `@ManyToMany` almost never warrants `CascadeType.REMOVE`; at most `PERSIST`/`MERGE`.

**F6.** How would you test with `@DataJpaTest` that removing an Appointment from a Patient's collection actually deletes it from the DB, not just the in-memory list?

> **Answer:** Persist a patient with one appointment, capture its ID, remove it from the collection, call `saveAndFlush()`, then `entityManager.clear()` (via `TestEntityManager`) to force a fresh DB read, and assert `appointmentRepository.findById(apptId)` is empty — proving the row is gone from the DB, not just absent from a stale in-memory reference.

**F7.** What happens if `Patient.insurance` only has `cascade = CascadeType.PERSIST` (no `MERGE`), and you try to UPDATE an existing patient's insurance details?

> **Answer:** `PERSIST` only handles "parent saved for the FIRST TIME with a new child." Updating an EXISTING patient's insurance without `MERGE` means Hibernate does NOT propagate the update automatically — changes may silently not save, or (if a NEW transient Insurance is assigned) throw `TransientPropertyValueException` on flush. The practical rule: cascade both `PERSIST` and `MERGE` together for lifecycle-following relationships.

**F8.** In a microservices architecture, why can't you simply "cascade" a delete across service boundaries (e.g., Patient Service deleting a patient should clean up Billing Service records)?

> **Answer:** JPA cascading only works within a SINGLE database/persistence context managed by ONE `EntityManager` — it has zero visibility into another service's database. Cross-service cleanup needs either synchronous calls (with Saga/compensating-transaction logic, since there's no true distributed ACID transaction) or asynchronous event-driven cleanup (publish a `PatientDeletedEvent`; Billing Service consumes it and cleans up independently, accepting eventual consistency).

```
