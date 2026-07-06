# Chapter 08 — QA: JPA Mappings, Hospital Management System

---

## Part A — Short Answer Questions (20)

**A1.** What are the four types of JPA mapping annotations? Give the SQL structure each creates.

> **Answer:**
> - `@OneToOne` → FK column (UNIQUE) on owning side
> - `@ManyToOne` → FK column on owning side
> - `@OneToMany` → usually no column (uses `mappedBy`); without `mappedBy` creates a join table
> - `@ManyToMany` → always creates a separate join table

---

**A2.** What is the Owning Side in a JPA relationship?

> **Answer:** The owning side is the entity that controls the foreign key updates in the database. It is identified by having `@JoinColumn` (for OneToOne/ManyToOne) or `@ManyToMany` without `mappedBy`. Only updates made through the owning side are persisted to the FK column.

---

**A3.** What is `mappedBy` and where do you put it?

> **Answer:** `mappedBy` is placed on the inverse side of a bidirectional relationship. Its value is the **field name** of the corresponding relationship in the owning entity. It tells JPA: "Don't create a new column here — the FK is already managed by the other side."
> Example: `@OneToMany(mappedBy = "patient")` where `"patient"` is the field name in `Appointment`.

---

**A4.** What happens if you write `@OneToMany` on the Patient side WITHOUT `mappedBy`?

> **Answer:** Hibernate creates a **join table** (e.g., `patient_appointments`) with `patient_id` and `appointment_id` columns. This is usually wrong for 1:N relationships. The correct approach is to use `@OneToMany(mappedBy = "patient")` which uses the existing FK in the Appointment table.

---

**A5.** Why does `@OneToOne` automatically add a UNIQUE constraint to the FK column?

> **Answer:** Because OneToOne means each insurance can be associated with only one patient and vice versa. Without UNIQUE, two patients could reference the same insurance, making it effectively OneToMany. Hibernate adds UNIQUE automatically to enforce the cardinality.

---

**A6.** Where is the foreign key column created for a `@ManyToOne` relationship?

> **Answer:** The FK column is created in the table of the entity that has the `@ManyToOne` annotation (the owning side). For example, `@ManyToOne` in `Appointment` creates `patient_id` in the `appointment` table — not in the `patient` table.

---

**A7.** What is a join table in the context of ManyToMany mapping? What columns does it have?

> **Answer:** A join table is an intermediate table created by Hibernate to represent a ManyToMany relationship. It has two columns — one FK for each entity. Example: `department_doctors` table has `department_id` (FK → department) and `doctor_id` (FK → doctor). Both columns together form the composite primary key.

---

**A8.** Can two entities have more than one relationship between them? Give an example.

> **Answer:** Yes. In the example, `Department` and `Doctor` have two relationships:
> 1. `@OneToOne headDoctor` — the department head
> 2. `@ManyToMany doctors` — all doctors in the department
> Each relationship creates a separate FK column or join table.

---

**A9.** Why must you initialize collection fields (`= new HashSet<>()`) in ManyToMany entities?

> **Answer:** If the collection is null and Hibernate tries to populate it when loading a relationship, a `NullPointerException` occurs. Initializing with `new HashSet<>()` ensures a non-null collection is always available.

---

**A10.** How do you customize the name of the join table and its columns in ManyToMany?

> **Answer:** Use `@JoinTable` on the owning side:
> ```java
> @ManyToMany
> @JoinTable(
>     name = "my_join_table",
>     joinColumns = @JoinColumn(name = "dept_id"),
>     inverseJoinColumns = @JoinColumn(name = "doc_id")
> )
> private Set<Doctor> doctors;
> ```

---

**A11.** What does the circle symbol mean in an ERD diagram?

> **Answer:** A circle indicates optional participation (also called partial participation). An entity with a circle can exist without being part of that relationship. Example: Patient with a circle on the insurance side means a patient can exist without insurance.

---

**A12.** What is the difference between unidirectional and bidirectional mapping?

> **Answer:**
> - **Unidirectional:** Only one entity knows about the other. E.g., Patient has `insurance` field, but Insurance has no `patient` field. Navigation is one-way only.
> - **Bidirectional:** Both entities know about each other. E.g., Patient has `insurance` and Insurance has `patient`. Navigation works from both sides. Requires `mappedBy` on the inverse side.

---

**A13.** How does Hibernate determine the default column name for a join column when `@JoinColumn` is not specified?

> **Answer:** Hibernate uses the convention: `fieldName_referencedColumnName`. For a field `insurance` referencing an entity with PK `id`, the column becomes `insurance_id`.

---

**A14.** In an ERD diagram, what does a "crow's foot" notation represent?

> **Answer:** A crow's foot (arrow/fork symbol) represents the "many" side of a relationship. A single straight bar `|` represents the "one" side. Together `|────<` means "one to many".

---

**A15.** What is the "Single Source of Truth" principle in JPA relationships?

> **Answer:** Only one entity (the owning side) should manage the foreign key value. Having two entities both claim to update the same FK creates ambiguity — Hibernate can't know which is the authoritative value. `mappedBy` resolves this by designating one side as authoritative.

---

**A16.** If you update an appointment's patient list via `patient.getAppointments().add(appt)` and save the patient, will the DB be updated?

> **Answer:** No. `Patient` is the inverse side for the Patient-Appointment relationship. The FK `patient_id` is owned by `Appointment`. To update the DB, you must set `appt.setPatient(patient)` and save the appointment (owning side).

---

**A17.** Write the `Doctor` entity field declarations for both department relationships (head and member).

> **Answer:**
> ```java
> // Inverse side of OneToOne (Department -> headDoctor)
> @OneToOne(mappedBy = "headDoctor")
> private Department headOf;
>
> // Inverse side of ManyToMany (Department -> doctors)
> @ManyToMany(mappedBy = "doctors")
> private Set<Department> departments = new HashSet<>();
> ```

---

**A18.** What is JPA Buddy? How does it help in IntelliJ IDEA?

> **Answer:** JPA Buddy is an IntelliJ plugin (install via Preferences → Plugins → Marketplace) that auto-generates Spring Data repository interfaces, JPQL queries, and DDL scripts from entity classes. It reads entity fields and ID type automatically, saving boilerplate coding.

---

**A19.** What does `nullable = false` on `@JoinColumn` enforce?

> **Answer:** It adds a NOT NULL constraint to the FK column in the database. This means the relationship is mandatory — the owning entity cannot be saved without the referenced entity. Example: Appointment must always have a Patient.

---

**A20.** In a ManyToMany join table, why are both FKs also the composite primary key?

> **Answer:** In a join table, there is no standalone ID. The combination of both FK values (e.g., `department_id + doctor_id`) uniquely identifies each row — meaning a specific (department, doctor) pair can appear only once. This pair forms the composite primary key.

---

## Part B — Multiple Choice Questions (12)

**B1.** Where is the FK column created for `@OneToOne` on the Patient side?

- a) In the insurance table
- b) In the patient table ✅
- c) In a new join table
- d) No FK is created

---

**B2.** Which annotation marks the inverse side of a bidirectional relationship?

- a) `@JoinColumn`
- b) `@InverseJoin`
- c) `mappedBy` ✅
- d) `@Bidirectional`

---

**B3.** What does `@OneToMany` without `mappedBy` create?

- a) A FK column in the parent table
- b) A FK column in the child table
- c) A join table ✅
- d) Nothing — it's a compile error

---

**B4.** Which entity is the owning side in Patient-Appointment (OneToMany/ManyToOne)?

- a) Patient (has @OneToMany)
- b) Appointment (has @ManyToOne) ✅
- c) Both entities equally own it
- d) Neither — JPA decides automatically

---

**B5.** What must you add to Doctor if you want a bidirectional ManyToMany with Department?

- a) `@ManyToMany` and a `Set<Department>` with `mappedBy` ✅
- b) `@OneToMany` with a `List<Department>`
- c) `@JoinTable` with column definitions
- d) Nothing — bidirectional is automatic

---

**B6.** What SQL constraint is automatically added to the FK column in a `@OneToOne` relationship?

- a) NOT NULL
- b) FOREIGN KEY
- c) UNIQUE ✅
- d) CHECK

---

**B7.** How do you customize the join table name in `@ManyToMany`?

- a) `@JoinColumn(name = "...")`
- b) `@Table(name = "...")`
- c) `@JoinTable(name = "...")` ✅
- d) `@ManyToMany(tableName = "...")`

---

**B8.** Which side controls the FK update when you save a new appointment?

- a) Patient (inverse side)
- b) Appointment (owning side) ✅
- c) Both sides simultaneously
- d) Neither — JPA auto-detects

---

**B9.** In a join table for ManyToMany, what forms the primary key?

- a) A separate auto-generated ID column
- b) One of the FKs
- c) The composite pair of both FKs ✅
- d) There is no primary key in a join table

---

**B10.** What is the value of `mappedBy` in `@OneToMany(mappedBy = "patient")` on the Patient entity?

- a) The class name "Appointment"
- b) The table name "appointment"
- c) The field name in Appointment entity ✅
- d) The repository bean name

---

**B11.** What happens when you try to save a new Appointment entity with `patient` field as null, given `@JoinColumn(nullable = false)`?

- a) The appointment is saved with a null FK
- b) The application ignores the null value
- c) A DB constraint violation exception is thrown ✅
- d) Spring Boot auto-generates a default patient

---

**B12.** What is the `Set<Doctor> doctors = new HashSet<>()` initialization for?

- a) To pre-populate the collection with all doctors
- b) To prevent NullPointerException when Hibernate populates the collection ✅
- c) It has no effect — Hibernate replaces it anyway
- d) Required by the JPA spec for Set types

---

## Part C — Scenario and Code Questions (10)

**C1.** Write the complete `Insurance` entity with fields: `id`, `policyNumber`, `provider`, `validUntil`, `createdAt`. Include the bidirectional relationship back to `Patient`.

> **Answer:**
> ```java
> @Entity
> @Getter @Setter @NoArgsConstructor @AllArgsConstructor @ToString
> public class Insurance {
>     @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
>     private Long id;
>     private String policyNumber;
>     private String provider;
>     private LocalDate validUntil;
>     @CreationTimestamp
>     private LocalDateTime createdAt;
>
>     @OneToOne(mappedBy = "insurance")
>     @ToString.Exclude      // avoid circular toString
>     private Patient patient;
> }
> ```

---

**C2.** A developer writes this code. Will the appointment's patient be saved to the DB? Why?

> ```java
> patient.getAppointments().add(appointment);
> patientRepository.save(patient);
> ```

> **Answer:** No. `Patient.appointments` is the inverse side (has `mappedBy`). Updating the inverse side collection does NOT update the `patient_id` FK in the `appointment` table. The correct way is:
> ```java
> appointment.setPatient(patient);
> appointmentRepository.save(appointment);
> ```

---

**C3.** Write the `Appointment` entity showing both ManyToOne relationships (to Patient and Doctor) with nullable=false for both.

> **Answer:**
> ```java
> @Entity
> @Getter @Setter @NoArgsConstructor @AllArgsConstructor
> public class Appointment {
>     @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
>     private Long id;
>     private LocalDateTime appointmentTime;
>     private String reason;
>
>     @ManyToOne
>     @JoinColumn(name = "patient_id", nullable = false)
>     private Patient patient;
>
>     @ManyToOne
>     @JoinColumn(name = "doctor_id", nullable = false)
>     private Doctor doctor;
> }
> ```

---

**C4.** Write the `Department` entity with a department-head `@OneToOne` and an all-doctors `@ManyToMany`, both pointing to `Doctor`.

> **Answer:**
> ```java
> @Entity
> @Getter @Setter @NoArgsConstructor @AllArgsConstructor
> public class Department {
>     @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
>     private Long id;
>
>     @Column(unique = true)
>     private String name;
>
>     @OneToOne
>     private Doctor headDoctor;
>
>     @ManyToMany
>     @JoinTable(
>         name = "department_doctors",
>         joinColumns = @JoinColumn(name = "department_id"),
>         inverseJoinColumns = @JoinColumn(name = "doctor_id")
>     )
>     private Set<Doctor> doctors = new HashSet<>();
> }
> ```

---

**C5.** What SQL tables and columns does Hibernate create for the above Department entity?

> **Answer:**
> ```sql
> CREATE TABLE department (
>     id BIGINT PRIMARY KEY,
>     name VARCHAR UNIQUE,
>     head_doctor_id BIGINT UNIQUE REFERENCES doctor(id)
> );
>
> CREATE TABLE department_doctors (
>     department_id BIGINT REFERENCES department(id),
>     doctor_id     BIGINT REFERENCES doctor(id),
>     PRIMARY KEY (department_id, doctor_id)
> );
> ```

---

**C6.** Write the code to add Doctor "Dr. Smith" to Department "Cardiology" and persist it.

> **Answer:**
> ```java
> Doctor smith = doctorRepository.findByName("Dr. Smith");
> Department cardiology = departmentRepository.findByName("Cardiology");
>
> cardiology.getDoctors().add(smith);    // owning side
> departmentRepository.save(cardiology); // persists to join table ✅
> ```
> Note: Do NOT add via `smith.getDepartments().add(cardiology)` — that's the inverse side and won't update the join table.

---

**C7.** Explain what ERD notation `Patient O──|──<── Appointment` means.

> **Answer:**
> - `O` = Patient has optional participation (can exist without an appointment)
> - `|` on Patient side = "one" Patient
> - `<` on Appointment side = "many" Appointments
> - Reading: One Patient can have many Appointments. A Patient can exist with zero appointments. One Appointment must belong to exactly one Patient.

---

**C8.** You write `@OneToMany` on Patient without `mappedBy`. Run the app and see a new table `patient_appointments`. What went wrong and how do you fix it?

> **Answer:** Without `mappedBy`, Hibernate treats the @OneToMany as the owning side and creates a join table. Fix:
> ```java
> // Patient.java
> @OneToMany(mappedBy = "patient")  // add mappedBy referencing field in Appointment
> private List<Appointment> appointments;
> ```
> Now Hibernate uses the `patient_id` FK in the `appointment` table instead of a join table.

---

**C9.** How do you set a custom FK column name `ins_id` for the Patient-Insurance join column?

> **Answer:**
> ```java
> @OneToOne
> @JoinColumn(name = "ins_id")
> private Insurance insurance;
> ```

---

**C10.** A new developer adds the line `@ToString` on `Patient` and `Insurance` with bidirectional mapping. What runtime error could occur?

> **Answer:** `StackOverflowError` due to infinite recursion. `Patient.toString()` calls `Insurance.toString()` which calls `Patient.toString()` and so on forever.
> Fix: Use `@ToString.Exclude` on the relationship field on the inverse side:
> ```java
> // In Insurance
> @OneToOne(mappedBy = "insurance")
> @ToString.Exclude
> private Patient patient;
> ```

---

## Part D — Fill in the Blanks (8)

**D1.** In a `@OneToOne` relationship, Hibernate automatically adds a ______ constraint to the FK column.

> **Answer:** UNIQUE

---

**D2.** `mappedBy` value must match the ______ name in the owning entity, not the class name.

> **Answer:** field

---

**D3.** In `@ManyToMany`, Hibernate creates a ______ table containing FK columns for both entities.

> **Answer:** join

---

**D4.** The entity that has `@JoinColumn` is called the ______ side of the relationship.

> **Answer:** owning

---

**D5.** If you modify a collection on the ______ side only, the FK in the database will NOT be updated.

> **Answer:** inverse

---

**D6.** Collection fields in ManyToMany should always be initialized with `= new ______()` to avoid NullPointerException.

> **Answer:** `HashSet<>` (or `ArrayList<>`)

---

**D7.** In an ERD diagram, a ______ symbol on an entity's side of a relationship means that entity can exist without participating in the relationship.

> **Answer:** circle (O)

---

**D8.** When two entities have a ManyToMany relationship, both FK columns in the join table also serve as the ______ key.

> **Answer:** primary (composite primary key)

---

## Part E — True or False (8)

**E1.** In a bidirectional OneToOne, both entities get a FK column in their respective tables.

> **Answer:** False — only the owning side gets the FK column. The inverse side uses `mappedBy` and has no additional column.

---

**E2.** `@ManyToOne` can only be placed on the parent entity.

> **Answer:** False — `@ManyToOne` is placed on the CHILD entity (Appointment, not Patient). "Many Appointments to One Patient" — the annotation is on the "Many" side.

---

**E3.** Updating a collection on the inverse side and saving that entity will update the FK in the database.

> **Answer:** False — only owning-side updates are persisted to the FK column.

---

**E4.** Two different relationships between the same two entities are allowed in JPA.

> **Answer:** True — Department can have both a `@OneToOne headDoctor` and a `@ManyToMany doctors` pointing to Doctor.

---

**E5.** Without `mappedBy` on a `@OneToMany`, Hibernate creates a join table instead of using the FK on the child table.

> **Answer:** True.

---

**E6.** `@JoinTable` can be used to specify a custom name for the join table in ManyToMany mapping.

> **Answer:** True — `@JoinTable(name = "my_custom_table", ...)` on the owning side.

---

**E7.** In a ManyToMany join table, Hibernate always adds a separate `id` auto-generated primary key column.

> **Answer:** False — The join table uses a composite primary key formed by both FK columns. There is no separate auto-generated ID.

---

**E8.** The `mappedBy` attribute value is case-sensitive and must exactly match the Java field name in the owning entity.

> **Answer:** True — if the field is `insurance`, `mappedBy = "insurance"` not `"Insurance"` or `"INSURANCE"`.

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

**F1.** What is `MultipleBagFetchException`, and how does it relate to `List` vs `Set` for `@OneToMany`/`@ManyToMany`?

> **Answer:** If an entity has TWO OR MORE `List`-typed collections ("bags" in Hibernate terms) that are BOTH eagerly join-fetched in the same query, Hibernate can't cartesian-join two bags unambiguously and throws `MultipleBagFetchException`. Fixes: change one/both to `Set` (Hibernate CAN join-fetch multiple Sets with `DISTINCT`), fetch them in separate queries, or use `@BatchSize` instead.

**F2.** Why is `Set` generally preferred over `List` for `@ManyToMany`, beyond just avoiding `MultipleBagFetchException`?

> **Answer:** `List` requires Hibernate to track POSITION, so removing an element from the middle can trigger DELETE-then-REINSERT of ALL subsequent rows to preserve index order — an expensive surprise. `Set` has no positional semantics, so removing one element deletes just that one join-table row. Use `@OrderColumn` on a `List` only when true ordering is genuinely required.

**F3.** How do you model a self-referencing hierarchy (an `Employee` with one `Manager`, who is also an `Employee`, plus a `List<Employee> directReports`)?

> **Answer:** A self-referencing `@ManyToOne`/`@OneToMany` pair on the SAME entity: `manager` field with `@ManyToOne @JoinColumn(name="manager_id")` (owning side), and `directReports` with `@OneToMany(mappedBy = "manager")` (inverse side) — functionally identical to any other bidirectional 1:N mapping, just pointing to the same entity class on both ends.

**F4.** What is `@MapsId`, and when would you use it instead of a normal `@OneToOne` with its own generated `@Id`?

> **Answer:** `@MapsId` implements a shared primary key association — the child's `@Id` IS the same value as the parent's `@Id` (no separate FK column). Common for "extension table" patterns (e.g., `EmployeeProfile.id` literally equals `Employee.id`), guaranteeing true 1:1 cardinality at the schema level and saving a column.

**F5.** What are the two annotation-based approaches for a composite (multi-column) primary key in JPA?

> **Answer:** (1) `@EmbeddedId` — a separate `@Embeddable` class holding the composite fields, referenced as the entity's single `@Id` field. (2) `@IdClass` — a plain class mirroring the entity's individually `@Id`-annotated fields (matching names/types), used by Hibernate for identity without embedding. `@EmbeddedId` is generally preferred for its object-oriented reusability.

**F6.** Why might you deliberately keep a mapping UNIDIRECTIONAL even when bidirectional navigation "might be useful someday"?

> **Answer:** Bidirectional mappings add complexity — both sides must stay consistent in application code, they increase infinite-recursion/`MultipleBagFetchException` risk, and couple two entities more tightly. YAGNI applies: if the inverse navigation is never actually queried, adding it is pure complexity with no benefit. Add it only when a concrete use case requires navigating from the inverse side.

**F7.** What SQL-level bug can occur if you forget `@OrderBy` on a `List`-typed `@OneToMany`, and application code assumes a specific order?

> **Answer:** Without `@OrderBy`, collection iteration order is UNDEFINED (dependent on physical storage/query planner, not insertion order) and can change unpredictably. Code assuming `appointments.get(0)` is "the first/most recent" is a latent bug that may pass in dev (small, stable data) and fail in production. Fix: explicit `@OrderBy("appointmentTime DESC")`, or sort at the query level.

**F8.** A `@ManyToMany` relationship needs an extra attribute on the relationship itself (e.g., `assignedDate`). Can plain `@ManyToMany` express this?

> **Answer:** No — plain `@ManyToMany` only supports a simple join table with the two FK columns. The fix is to "promote" the join table to its own entity (e.g., `DepartmentDoctorAssignment`) with `@ManyToOne` to both sides PLUS the extra columns — converting one `@ManyToMany` into two `@OneToMany`/`@ManyToOne` pairs through an explicit join entity, a common real-world modeling upgrade.

```
