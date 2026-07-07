# Chapter 08 — QA: JPA Mappings, Hospital Management System

---

## Part A — Short Answer Questions (20)

<a id="qA1"></a>**A1.** What are the four types of JPA mapping annotations? Give the SQL structure each creates. [↓ Answer](#aA1)

<a id="aA1"></a>> **Answer:**
>
> - `@OneToOne` → FK column (UNIQUE) on owning side
> - `@ManyToOne` → FK column on owning side
> - `@OneToMany` → usually no column (uses `mappedBy`); without `mappedBy` creates a join table
> - `@ManyToMany` → always creates a separate join table
>
> [↑ Question](#qA1)

---

<a id="qA2"></a>**A2.** What is the Owning Side in a JPA relationship? [↓ Answer](#aA2)

<a id="aA2"></a>> **Answer:** The owning side is the entity that controls the foreign key updates in the database. It is identified by having `@JoinColumn` (for OneToOne/ManyToOne) or `@ManyToMany` without `mappedBy`. Only updates made through the owning side are persisted to the FK column. [↑ Question](#qA2)

---

<a id="qA3"></a>**A3.** What is `mappedBy` and where do you put it? [↓ Answer](#aA3)

<a id="aA3"></a>> **Answer:** `mappedBy` is placed on the inverse side of a bidirectional relationship. Its value is the **field name** of the corresponding relationship in the owning entity. It tells JPA: "Don't create a new column here — the FK is already managed by the other side."
> Example: `@OneToMany(mappedBy = "patient")` where `"patient"` is the field name in `Appointment`. [↑ Question](#qA3)

---

<a id="qA4"></a>**A4.** What happens if you write `@OneToMany` on the Patient side WITHOUT `mappedBy`? [↓ Answer](#aA4)

<a id="aA4"></a>> **Answer:** Hibernate creates a **join table** (e.g., `patient_appointments`) with `patient_id` and `appointment_id` columns. This is usually wrong for 1:N relationships. The correct approach is to use `@OneToMany(mappedBy = "patient")` which uses the existing FK in the Appointment table. [↑ Question](#qA4)

---

<a id="qA5"></a>**A5.** Why does `@OneToOne` automatically add a UNIQUE constraint to the FK column? [↓ Answer](#aA5)

<a id="aA5"></a>> **Answer:** Because OneToOne means each insurance can be associated with only one patient and vice versa. Without UNIQUE, two patients could reference the same insurance, making it effectively OneToMany. Hibernate adds UNIQUE automatically to enforce the cardinality. [↑ Question](#qA5)

---

<a id="qA6"></a>**A6.** Where is the foreign key column created for a `@ManyToOne` relationship? [↓ Answer](#aA6)

<a id="aA6"></a>> **Answer:** The FK column is created in the table of the entity that has the `@ManyToOne` annotation (the owning side). For example, `@ManyToOne` in `Appointment` creates `patient_id` in the `appointment` table — not in the `patient` table. [↑ Question](#qA6)

---

<a id="qA7"></a>**A7.** What is a join table in the context of ManyToMany mapping? What columns does it have? [↓ Answer](#aA7)

<a id="aA7"></a>> **Answer:** A join table is an intermediate table created by Hibernate to represent a ManyToMany relationship. It has two columns — one FK for each entity. Example: `department_doctors` table has `department_id` (FK → department) and `doctor_id` (FK → doctor). Both columns together form the composite primary key. [↑ Question](#qA7)

---

<a id="qA8"></a>**A8.** Can two entities have more than one relationship between them? Give an example. [↓ Answer](#aA8)

<a id="aA8"></a>> **Answer:** Yes. In the example, `Department` and `Doctor` have two relationships:
>
> 1. `@OneToOne headDoctor` — the department head
> 2. `@ManyToMany doctors` — all doctors in the department
>
> Each relationship creates a separate FK column or join table. [↑ Question](#qA8)

---

<a id="qA9"></a>**A9.** Why must you initialize collection fields (`= new HashSet<>()`) in ManyToMany entities? [↓ Answer](#aA9)

<a id="aA9"></a>> **Answer:** If the collection is null and Hibernate tries to populate it when loading a relationship, a `NullPointerException` occurs. Initializing with `new HashSet<>()` ensures a non-null collection is always available. [↑ Question](#qA9)

---

<a id="qA10"></a>**A10.** How do you customize the name of the join table and its columns in ManyToMany? [↓ Answer](#aA10)

<a id="aA10"></a>> **Answer:** Use `@JoinTable` on the owning side:
>
> ```java
> @ManyToMany
> @JoinTable(
>     name = "my_join_table",
>     joinColumns = @JoinColumn(name = "dept_id"),
>     inverseJoinColumns = @JoinColumn(name = "doc_id")
> )
> private Set<Doctor> doctors;
> ```
>
> [↑ Question](#qA10)

---

<a id="qA11"></a>**A11.** What does the circle symbol mean in an ERD diagram? [↓ Answer](#aA11)

<a id="aA11"></a>> **Answer:** A circle indicates optional participation (also called partial participation). An entity with a circle can exist without being part of that relationship. Example: Patient with a circle on the insurance side means a patient can exist without insurance. [↑ Question](#qA11)

---

<a id="qA12"></a>**A12.** What is the difference between unidirectional and bidirectional mapping? [↓ Answer](#aA12)

<a id="aA12"></a>> **Answer:**
>
> - **Unidirectional:** Only one entity knows about the other. E.g., Patient has `insurance` field, but Insurance has no `patient` field. Navigation is one-way only.
> - **Bidirectional:** Both entities know about each other. E.g., Patient has `insurance` and Insurance has `patient`. Navigation works from both sides. Requires `mappedBy` on the inverse side.
>
> [↑ Question](#qA12)

---

<a id="qA13"></a>**A13.** How does Hibernate determine the default column name for a join column when `@JoinColumn` is not specified? [↓ Answer](#aA13)

<a id="aA13"></a>> **Answer:** Hibernate uses the convention: `fieldName_referencedColumnName`. For a field `insurance` referencing an entity with PK `id`, the column becomes `insurance_id`. [↑ Question](#qA13)

---

<a id="qA14"></a>**A14.** In an ERD diagram, what does a "crow's foot" notation represent? [↓ Answer](#aA14)

<a id="aA14"></a>> **Answer:** A crow's foot (arrow/fork symbol) represents the "many" side of a relationship. A single straight bar `|` represents the "one" side. Together `|────<` means "one to many". [↑ Question](#qA14)

---

<a id="qA15"></a>**A15.** What is the "Single Source of Truth" principle in JPA relationships? [↓ Answer](#aA15)

<a id="aA15"></a>> **Answer:** Only one entity (the owning side) should manage the foreign key value. Having two entities both claim to update the same FK creates ambiguity — Hibernate can't know which is the authoritative value. `mappedBy` resolves this by designating one side as authoritative. [↑ Question](#qA15)

---

<a id="qA16"></a>**A16.** If you update an appointment's patient list via `patient.getAppointments().add(appt)` and save the patient, will the DB be updated? [↓ Answer](#aA16)

<a id="aA16"></a>> **Answer:** No. `Patient` is the inverse side for the Patient-Appointment relationship. The FK `patient_id` is owned by `Appointment`. To update the DB, you must set `appt.setPatient(patient)` and save the appointment (owning side). [↑ Question](#qA16)

---

<a id="qA17"></a>**A17.** Write the `Doctor` entity field declarations for both department relationships (head and member). [↓ Answer](#aA17)

<a id="aA17"></a>> **Answer:**
>
> ```java
> // Inverse side of OneToOne (Department -> headDoctor)
> @OneToOne(mappedBy = "headDoctor")
> private Department headOf;
>
> // Inverse side of ManyToMany (Department -> doctors)
> @ManyToMany(mappedBy = "doctors")
> private Set<Department> departments = new HashSet<>();
> ```
>
> [↑ Question](#qA17)

---

<a id="qA18"></a>**A18.** What is JPA Buddy? How does it help in IntelliJ IDEA? [↓ Answer](#aA18)

<a id="aA18"></a>> **Answer:** JPA Buddy is an IntelliJ plugin (install via Preferences → Plugins → Marketplace) that auto-generates Spring Data repository interfaces, JPQL queries, and DDL scripts from entity classes. It reads entity fields and ID type automatically, saving boilerplate coding. [↑ Question](#qA18)

---

<a id="qA19"></a>**A19.** What does `nullable = false` on `@JoinColumn` enforce? [↓ Answer](#aA19)

<a id="aA19"></a>> **Answer:** It adds a NOT NULL constraint to the FK column in the database. This means the relationship is mandatory — the owning entity cannot be saved without the referenced entity. Example: Appointment must always have a Patient. [↑ Question](#qA19)

---

<a id="qA20"></a>**A20.** In a ManyToMany join table, why are both FKs also the composite primary key? [↓ Answer](#aA20)

<a id="aA20"></a>> **Answer:** In a join table, there is no standalone ID. The combination of both FK values (e.g., `department_id + doctor_id`) uniquely identifies each row — meaning a specific (department, doctor) pair can appear only once. This pair forms the composite primary key. [↑ Question](#qA20)

---

## Part B — Multiple Choice Questions (12)

<a id="qB1"></a>**B1.** Where is the FK column created for `@OneToOne` on the Patient side? [↓ Answer](#aB1)

- a) In the insurance table
- b) <a id="aB1"></a>In the patient table ✅ [↑ Question](#qB1)
- c) In a new join table
- d) No FK is created

---

<a id="qB2"></a>**B2.** Which annotation marks the inverse side of a bidirectional relationship? [↓ Answer](#aB2)

- a) `@JoinColumn`
- b) `@InverseJoin`
- c) <a id="aB2"></a>`mappedBy` ✅ [↑ Question](#qB2)
- d) `@Bidirectional`

---

<a id="qB3"></a>**B3.** What does `@OneToMany` without `mappedBy` create? [↓ Answer](#aB3)

- a) A FK column in the parent table
- b) A FK column in the child table
- c) <a id="aB3"></a>A join table ✅ [↑ Question](#qB3)
- d) Nothing — it's a compile error

---

<a id="qB4"></a>**B4.** Which entity is the owning side in Patient-Appointment (OneToMany/ManyToOne)? [↓ Answer](#aB4)

- a) Patient (has @OneToMany)
- b) <a id="aB4"></a>Appointment (has @ManyToOne) ✅ [↑ Question](#qB4)
- c) Both entities equally own it
- d) Neither — JPA decides automatically

---

<a id="qB5"></a>**B5.** What must you add to Doctor if you want a bidirectional ManyToMany with Department? [↓ Answer](#aB5)

- a) <a id="aB5"></a>`@ManyToMany` and a `Set<Department>` with `mappedBy` ✅ [↑ Question](#qB5)
- b) `@OneToMany` with a `List<Department>`
- c) `@JoinTable` with column definitions
- d) Nothing — bidirectional is automatic

---

<a id="qB6"></a>**B6.** What SQL constraint is automatically added to the FK column in a `@OneToOne` relationship? [↓ Answer](#aB6)

- a) NOT NULL
- b) FOREIGN KEY
- c) <a id="aB6"></a>UNIQUE ✅ [↑ Question](#qB6)
- d) CHECK

---

<a id="qB7"></a>**B7.** How do you customize the join table name in `@ManyToMany`? [↓ Answer](#aB7)

- a) `@JoinColumn(name = "...")`
- b) `@Table(name = "...")`
- c) <a id="aB7"></a>`@JoinTable(name = "...")` ✅ [↑ Question](#qB7)
- d) `@ManyToMany(tableName = "...")`

---

<a id="qB8"></a>**B8.** Which side controls the FK update when you save a new appointment? [↓ Answer](#aB8)

- a) Patient (inverse side)
- b) <a id="aB8"></a>Appointment (owning side) ✅ [↑ Question](#qB8)
- c) Both sides simultaneously
- d) Neither — JPA auto-detects

---

<a id="qB9"></a>**B9.** In a join table for ManyToMany, what forms the primary key? [↓ Answer](#aB9)

- a) A separate auto-generated ID column
- b) One of the FKs
- c) <a id="aB9"></a>The composite pair of both FKs ✅ [↑ Question](#qB9)
- d) There is no primary key in a join table

---

<a id="qB10"></a>**B10.** What is the value of `mappedBy` in `@OneToMany(mappedBy = "patient")` on the Patient entity? [↓ Answer](#aB10)

- a) The class name "Appointment"
- b) The table name "appointment"
- c) <a id="aB10"></a>The field name in Appointment entity ✅ [↑ Question](#qB10)
- d) The repository bean name

---

<a id="qB11"></a>**B11.** What happens when you try to save a new Appointment entity with `patient` field as null, given `@JoinColumn(nullable = false)`? [↓ Answer](#aB11)

- a) The appointment is saved with a null FK
- b) The application ignores the null value
- c) <a id="aB11"></a>A DB constraint violation exception is thrown ✅ [↑ Question](#qB11)
- d) Spring Boot auto-generates a default patient

---

<a id="qB12"></a>**B12.** What is the `Set<Doctor> doctors = new HashSet<>()` initialization for? [↓ Answer](#aB12)

- a) To pre-populate the collection with all doctors
- b) <a id="aB12"></a>To prevent NullPointerException when Hibernate populates the collection ✅ [↑ Question](#qB12)
- c) It has no effect — Hibernate replaces it anyway
- d) Required by the JPA spec for Set types

---

## Part C — Scenario and Code Questions (10)

<a id="qC1"></a>**C1.** Write the complete `Insurance` entity with fields: `id`, `policyNumber`, `provider`, `validUntil`, `createdAt`. Include the bidirectional relationship back to `Patient`. [↓ Answer](#aC1)

<a id="aC1"></a>> **Answer:**
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
>
> [↑ Question](#qC1)

---

<a id="qC2"></a>**C2.** A developer writes this code. Will the appointment's patient be saved to the DB? Why? [↓ Answer](#aC2)

> ```java
> patient.getAppointments().add(appointment);
> patientRepository.save(patient);
> ```

<a id="aC2"></a>> **Answer:** No. `Patient.appointments` is the inverse side (has `mappedBy`). Updating the inverse side collection does NOT update the `patient_id` FK in the `appointment` table. The correct way is:
> ```java
> appointment.setPatient(patient);
> appointmentRepository.save(appointment);
> ```
>
> [↑ Question](#qC2)

---

<a id="qC3"></a>**C3.** Write the `Appointment` entity showing both ManyToOne relationships (to Patient and Doctor) with nullable=false for both. [↓ Answer](#aC3)

<a id="aC3"></a>> **Answer:**
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
>
> [↑ Question](#qC3)

---

<a id="qC4"></a>**C4.** Write the `Department` entity with a department-head `@OneToOne` and an all-doctors `@ManyToMany`, both pointing to `Doctor`. [↓ Answer](#aC4)

<a id="aC4"></a>> **Answer:**
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
>
> [↑ Question](#qC4)

---

<a id="qC5"></a>**C5.** What SQL tables and columns does Hibernate create for the above Department entity? [↓ Answer](#aC5)

<a id="aC5"></a>> **Answer:**
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
>
> [↑ Question](#qC5)

---

<a id="qC6"></a>**C6.** Write the code to add Doctor "Dr. Smith" to Department "Cardiology" and persist it. [↓ Answer](#aC6)

<a id="aC6"></a>> **Answer:**
> ```java
> Doctor smith = doctorRepository.findByName("Dr. Smith");
> Department cardiology = departmentRepository.findByName("Cardiology");
>
> cardiology.getDoctors().add(smith);    // owning side
> departmentRepository.save(cardiology); // persists to join table ✅
> ```
>
> Note: Do NOT add via `smith.getDepartments().add(cardiology)` — that's the inverse side and won't update the join table. [↑ Question](#qC6)

---

<a id="qC7"></a>**C7.** Explain what ERD notation `Patient O──|──<── Appointment` means. [↓ Answer](#aC7)

<a id="aC7"></a>> **Answer:**
>
> - `O` = Patient has optional participation (can exist without an appointment)
> - `|` on Patient side = "one" Patient
> - `<` on Appointment side = "many" Appointments
> - Reading: One Patient can have many Appointments. A Patient can exist with zero appointments. One Appointment must belong to exactly one Patient.
>
> [↑ Question](#qC7)

---

<a id="qC8"></a>**C8.** You write `@OneToMany` on Patient without `mappedBy`. Run the app and see a new table `patient_appointments`. What went wrong and how do you fix it? [↓ Answer](#aC8)

<a id="aC8"></a>> **Answer:** Without `mappedBy`, Hibernate treats the @OneToMany as the owning side and creates a join table. Fix:
> ```java
> // Patient.java
> @OneToMany(mappedBy = "patient")  // add mappedBy referencing field in Appointment
> private List<Appointment> appointments;
> ```
> Now Hibernate uses the `patient_id` FK in the `appointment` table instead of a join table. [↑ Question](#qC8)

---

<a id="qC9"></a>**C9.** How do you set a custom FK column name `ins_id` for the Patient-Insurance join column? [↓ Answer](#aC9)

<a id="aC9"></a>> **Answer:**
> ```java
> @OneToOne
> @JoinColumn(name = "ins_id")
> private Insurance insurance;
> ```
>
> [↑ Question](#qC9)

---

<a id="qC10"></a>**C10.** A new developer adds the line `@ToString` on `Patient` and `Insurance` with bidirectional mapping. What runtime error could occur? [↓ Answer](#aC10)

<a id="aC10"></a>> **Answer:** `StackOverflowError` due to infinite recursion. `Patient.toString()` calls `Insurance.toString()` which calls `Patient.toString()` and so on forever.
> Fix: Use `@ToString.Exclude` on the relationship field on the inverse side:
> ```java
> // In Insurance
> @OneToOne(mappedBy = "insurance")
> @ToString.Exclude
> private Patient patient;
> ```
>
> [↑ Question](#qC10)

---

## Part D — Fill in the Blanks (8)

<a id="qD1"></a>**D1.** In a `@OneToOne` relationship, Hibernate automatically adds a ______ constraint to the FK column. [↓ Answer](#aD1)

<a id="aD1"></a>> **Answer:** UNIQUE [↑ Question](#qD1)

---

<a id="qD2"></a>**D2.** `mappedBy` value must match the ______ name in the owning entity, not the class name. [↓ Answer](#aD2)

<a id="aD2"></a>> **Answer:** field [↑ Question](#qD2)

---

<a id="qD3"></a>**D3.** In `@ManyToMany`, Hibernate creates a ______ table containing FK columns for both entities. [↓ Answer](#aD3)

<a id="aD3"></a>> **Answer:** join [↑ Question](#qD3)

---

<a id="qD4"></a>**D4.** The entity that has `@JoinColumn` is called the ______ side of the relationship. [↓ Answer](#aD4)

<a id="aD4"></a>> **Answer:** owning [↑ Question](#qD4)

---

<a id="qD5"></a>**D5.** If you modify a collection on the ______ side only, the FK in the database will NOT be updated. [↓ Answer](#aD5)

<a id="aD5"></a>> **Answer:** inverse [↑ Question](#qD5)

---

<a id="qD6"></a>**D6.** Collection fields in ManyToMany should always be initialized with `= new ______()` to avoid NullPointerException. [↓ Answer](#aD6)

<a id="aD6"></a>> **Answer:** `HashSet<>` (or `ArrayList<>`) [↑ Question](#qD6)

---

<a id="qD7"></a>**D7.** In an ERD diagram, a ______ symbol on an entity's side of a relationship means that entity can exist without participating in the relationship. [↓ Answer](#aD7)

<a id="aD7"></a>> **Answer:** circle (O) [↑ Question](#qD7)

---

<a id="qD8"></a>**D8.** When two entities have a ManyToMany relationship, both FK columns in the join table also serve as the ______ key. [↓ Answer](#aD8)

<a id="aD8"></a>> **Answer:** primary (composite primary key) [↑ Question](#qD8)

---

## Part E — True or False (8)

<a id="qE1"></a>**E1.** In a bidirectional OneToOne, both entities get a FK column in their respective tables. [↓ Answer](#aE1)

<a id="aE1"></a>> **Answer:** False — only the owning side gets the FK column. The inverse side uses `mappedBy` and has no additional column. [↑ Question](#qE1)

---

<a id="qE2"></a>**E2.** `@ManyToOne` can only be placed on the parent entity. [↓ Answer](#aE2)

<a id="aE2"></a>> **Answer:** False — `@ManyToOne` is placed on the CHILD entity (Appointment, not Patient). "Many Appointments to One Patient" — the annotation is on the "Many" side. [↑ Question](#qE2)

---

<a id="qE3"></a>**E3.** Updating a collection on the inverse side and saving that entity will update the FK in the database. [↓ Answer](#aE3)

<a id="aE3"></a>> **Answer:** False — only owning-side updates are persisted to the FK column. [↑ Question](#qE3)

---

<a id="qE4"></a>**E4.** Two different relationships between the same two entities are allowed in JPA. [↓ Answer](#aE4)

<a id="aE4"></a>> **Answer:** True — Department can have both a `@OneToOne headDoctor` and a `@ManyToMany doctors` pointing to Doctor. [↑ Question](#qE4)

---

<a id="qE5"></a>**E5.** Without `mappedBy` on a `@OneToMany`, Hibernate creates a join table instead of using the FK on the child table. [↓ Answer](#aE5)

<a id="aE5"></a>> **Answer:** True. [↑ Question](#qE5)

---

<a id="qE6"></a>**E6.** `@JoinTable` can be used to specify a custom name for the join table in ManyToMany mapping. [↓ Answer](#aE6)

<a id="aE6"></a>> **Answer:** True — `@JoinTable(name = "my_custom_table", ...)` on the owning side. [↑ Question](#qE6)

---

<a id="qE7"></a>**E7.** In a ManyToMany join table, Hibernate always adds a separate `id` auto-generated primary key column. [↓ Answer](#aE7)

<a id="aE7"></a>> **Answer:** False — The join table uses a composite primary key formed by both FK columns. There is no separate auto-generated ID. [↑ Question](#qE7)

---

<a id="qE8"></a>**E8.** The `mappedBy` attribute value is case-sensitive and must exactly match the Java field name in the owning entity. [↓ Answer](#aE8)

<a id="aE8"></a>> **Answer:** True — if the field is `insurance`, `mappedBy = "insurance"` not `"Insurance"` or `"INSURANCE"`. [↑ Question](#qE8)

---

## Part F — 🚀 Advanced / Real-World Interview Questions (Product-Company Level)

> These go beyond the transcript — the kind of follow-up/curveball questions asked at senior or product-company interviews.

<a id="qf1"></a>**F1.** What is `MultipleBagFetchException`, and how does it relate to `List` vs `Set` for `@OneToMany`/`@ManyToMany`? [↓ Answer](#af1)

<a id="af1"></a>> **Answer:** If an entity has TWO OR MORE `List`-typed collections ("bags" in Hibernate terms) that are BOTH eagerly join-fetched in the same query, Hibernate can't cartesian-join two bags unambiguously and throws `MultipleBagFetchException`. Fixes: change one/both to `Set` (Hibernate CAN join-fetch multiple Sets with `DISTINCT`), fetch them in separate queries, or use `@BatchSize` instead. [↑ Question](#qf1)

<a id="qf2"></a>**F2.** Why is `Set` generally preferred over `List` for `@ManyToMany`, beyond just avoiding `MultipleBagFetchException`? [↓ Answer](#af2)

<a id="af2"></a>> **Answer:** `List` requires Hibernate to track POSITION, so removing an element from the middle can trigger DELETE-then-REINSERT of ALL subsequent rows to preserve index order — an expensive surprise. `Set` has no positional semantics, so removing one element deletes just that one join-table row. Use `@OrderColumn` on a `List` only when true ordering is genuinely required. [↑ Question](#qf2)

<a id="qf3"></a>**F3.** How do you model a self-referencing hierarchy (an `Employee` with one `Manager`, who is also an `Employee`, plus a `List<Employee> directReports`)? [↓ Answer](#af3)

<a id="af3"></a>> **Answer:** A self-referencing `@ManyToOne`/`@OneToMany` pair on the SAME entity: `manager` field with `@ManyToOne @JoinColumn(name="manager_id")` (owning side), and `directReports` with `@OneToMany(mappedBy = "manager")` (inverse side) — functionally identical to any other bidirectional 1:N mapping, just pointing to the same entity class on both ends. [↑ Question](#qf3)

<a id="qf4"></a>**F4.** What is `@MapsId`, and when would you use it instead of a normal `@OneToOne` with its own generated `@Id`? [↓ Answer](#af4)

<a id="af4"></a>> **Answer:** `@MapsId` implements a shared primary key association — the child's `@Id` IS the same value as the parent's `@Id` (no separate FK column). Common for "extension table" patterns (e.g., `EmployeeProfile.id` literally equals `Employee.id`), guaranteeing true 1:1 cardinality at the schema level and saving a column. [↑ Question](#qf4)

<a id="qf5"></a>**F5.** What are the two annotation-based approaches for a composite (multi-column) primary key in JPA? [↓ Answer](#af5)

<a id="af5"></a>> **Answer:** (1) `@EmbeddedId` — a separate `@Embeddable` class holding the composite fields, referenced as the entity's single `@Id` field. (2) `@IdClass` — a plain class mirroring the entity's individually `@Id`-annotated fields (matching names/types), used by Hibernate for identity without embedding. `@EmbeddedId` is generally preferred for its object-oriented reusability. [↑ Question](#qf5)

<a id="qf6"></a>**F6.** Why might you deliberately keep a mapping UNIDIRECTIONAL even when bidirectional navigation "might be useful someday"? [↓ Answer](#af6)

<a id="af6"></a>> **Answer:** Bidirectional mappings add complexity — both sides must stay consistent in application code, they increase infinite-recursion/`MultipleBagFetchException` risk, and couple two entities more tightly. YAGNI applies: if the inverse navigation is never actually queried, adding it is pure complexity with no benefit. Add it only when a concrete use case requires navigating from the inverse side. [↑ Question](#qf6)

<a id="qf7"></a>**F7.** What SQL-level bug can occur if you forget `@OrderBy` on a `List`-typed `@OneToMany`, and application code assumes a specific order? [↓ Answer](#af7)

<a id="af7"></a>> **Answer:** Without `@OrderBy`, collection iteration order is UNDEFINED (dependent on physical storage/query planner, not insertion order) and can change unpredictably. Code assuming `appointments.get(0)` is "the first/most recent" is a latent bug that may pass in dev (small, stable data) and fail in production. Fix: explicit `@OrderBy("appointmentTime DESC")`, or sort at the query level. [↑ Question](#qf7)

<a id="qf8"></a>**F8.** A `@ManyToMany` relationship needs an extra attribute on the relationship itself (e.g., `assignedDate`). Can plain `@ManyToMany` express this? [↓ Answer](#af8)

<a id="af8"></a>> **Answer:** No — plain `@ManyToMany` only supports a simple join table with the two FK columns. The fix is to "promote" the join table to its own entity (e.g., `DepartmentDoctorAssignment`) with `@ManyToOne` to both sides PLUS the extra columns — converting one `@ManyToMany` into two `@OneToMany`/`@ManyToOne` pairs through an explicit join entity, a common real-world modeling upgrade. [↑ Question](#qf8)

```
