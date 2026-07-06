# Chapter 12 — JPA Mappings: Hospital Management System
**Timestamps:** `5:02:04 → 5:45:34`
**Topics:** ERD diagrams, OneToOne, OneToMany, ManyToOne, ManyToMany, Owning side, Inverse side, mappedBy, @JoinColumn, join tables, bidirectional mappings

---

## 1. Hospital Management System — Entities Overview

The project models a hospital with the following entities and relationships:

```
Patient ──(1:1)── Insurance
Patient ──(1:N)── Appointment
Doctor  ──(1:N)── Appointment
Department ──(1:1)── Doctor (head)
Department ──(N:N)── Doctor (staff)
```

**Entities created:**
- `Patient` — existing entity
- `Insurance` — patient's insurance policy
- `Appointment` — booking with a doctor
- `Doctor` — hospital doctor with specialization
- `Department` — hospital department (Cardiology, Physiotherapy, etc.)

---

## 2. ERD (Entity Relationship Diagram)

ERD notation used in the course:

| Symbol | Meaning |
|---|---|
| `|` (single bar) | One — exactly one |
| `<` (crow's foot) | Many — zero or more |
| `O` (circle) | Optional (partial participation) |
| No circle | Mandatory (full participation) |

**Example:**
```
Patient --|--O--<-- Appointment
          1       Many
```
- One patient → many appointments
- Circle on Patient side = Patient is optional in this notation (partial participation)
- No circle on Appointment = Appointment must have a Patient (full participation)

---

## 3. OneToOne Mapping — Patient ↔ Insurance

### 3.1 Unidirectional OneToOne

```java
// Patient.java (Owning Side)
@Entity
public class Patient {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToOne
    private Insurance insurance;   // Hibernate adds insurance_id FK column automatically
}
```

**What Hibernate generates:**
```sql
ALTER TABLE patient ADD COLUMN insurance_id BIGINT UNIQUE;
-- UNIQUE is added automatically for OneToOne (one insurance per patient)
```

### 3.2 Customizing the Join Column

```java
@OneToOne
@JoinColumn(name = "patient_insurance_id")   // custom FK column name
private Insurance insurance;
```

**`@JoinColumn` properties:**

| Property | Default | Description |
|---|---|---|
| `name` | `fieldName_id` | Column name in DB |
| `nullable` | `true` | Whether FK can be NULL |
| `unique` | `false` | (auto true for @OneToOne) |
| `referencedColumnName` | PK of referenced entity | Column in referenced table |
| `insertable` | `true` | Whether included in INSERT |
| `updatable` | `true` | Whether included in UPDATE |

**Patient can exist without insurance** (nullable=true, default).
Set `nullable=false` only if insurance is mandatory.

### 3.3 Bidirectional OneToOne

To navigate from Insurance → Patient as well:

```java
// Insurance.java (Inverse Side)
@Entity
public class Insurance {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String policyNumber;
    private String provider;
    private LocalDate validUntil;
    @CreationTimestamp
    private LocalDateTime createdAt;

    // Bidirectional — inverse side uses mappedBy
    @OneToOne(mappedBy = "insurance")   // "insurance" = field name in Patient
    private Patient patient;
}
```

> **`mappedBy`** tells JPA: "I am the inverse side. The owning side manages the FK. Don't create another column here."

Without `mappedBy`, Hibernate would also add `patient_id` to the insurance table → two FK columns → ambiguity → violation of single source of truth.

---

## 4. Owning Side vs Inverse Side

Every bidirectional relationship must have exactly one owning side and one inverse side.

| Concept | Description |
|---|---|
| **Owning Side** | Has `@JoinColumn`. Controls FK updates in DB. |
| **Inverse Side** | Has `mappedBy`. Read-only from JPA's perspective. |
| **Single Source of Truth** | Only one side should own the FK — prevents inconsistency |
| `mappedBy` value | Must match the **field name** in the owning entity (not class name) |

**Critical rule:** If you update data on the inverse side only, the DB will NOT be updated. Always save/update through the owning side.

```java
// CORRECT — update through owning side
appointment.setPatient(patient);
appointmentRepository.save(appointment);  // updates patient_id FK ✅

// WRONG — update through inverse side only
patient.getAppointments().add(appointment);
patientRepository.save(patient);  // does NOT update patient_id FK ❌
```

---

## 5. OneToMany / ManyToOne Mapping — Patient ↔ Appointment

**Rule:** One patient → many appointments; one appointment → exactly one patient.

### 5.1 Owning Side: Appointment (ManyToOne)

Appointment owns the relationship because it cannot exist without a patient.

```java
// Appointment.java (Owning Side)
@Entity
public class Appointment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private LocalDateTime appointmentTime;
    private String reason;

    @ManyToOne
    @JoinColumn(name = "patient_id", nullable = false)  // FK in appointment table
    private Patient patient;    // "many appointments to one patient"

    @ManyToOne
    @JoinColumn(nullable = false)    // doctor_id FK — auto-named
    private Doctor doctor;
}
```

### 5.2 Inverse Side: Patient (OneToMany)

```java
// Patient.java (Inverse Side)
@Entity
public class Patient {
    // ...
    @OneToMany(mappedBy = "patient")    // "patient" = field in Appointment
    private List<Appointment> appointments;
}
```

**Reading the annotation:**
- `@ManyToOne` on Appointment reads: "Many [Appointments] to One [Patient]"
- `@OneToMany` on Patient reads: "One [Patient] to Many [Appointments]"

**No extra column in Patient table.** The FK `patient_id` lives in `appointment` table only.

### 5.3 Without Bidirectional (@OneToMany without mappedBy)

If you write `@OneToMany` without `mappedBy`, Hibernate creates a **join table** instead of a FK column. This is almost always wrong for 1:N relationships.

```java
// BAD — creates a join table!
@OneToMany
private List<Appointment> appointments;

// GOOD — uses FK in appointment table
@OneToMany(mappedBy = "patient")
private List<Appointment> appointments;
```

---

## 6. ManyToMany Mapping — Department ↔ Doctor

One doctor can work in many departments. One department can have many doctors.

### 6.1 ManyToMany Requires a Join Table

```sql
-- Join table created automatically by Hibernate
CREATE TABLE department_doctors (
    department_id BIGINT REFERENCES department(id),  -- FK
    doctor_id     BIGINT REFERENCES doctor(id),      -- FK
    PRIMARY KEY (department_id, doctor_id)           -- composite PK
);
```

No separate `id` column — the pair (department_id, doctor_id) is the primary key.

### 6.2 Owning Side: Department

```java
// Department.java (Owning Side)
@Entity
public class Department {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String name;

    // OneToOne: department head
    @OneToOne
    private Doctor headDoctor;

    // ManyToMany: all doctors in this department
    @ManyToMany
    @JoinTable(
        name = "my_dpt_doctors",                        // custom join table name
        joinColumns = @JoinColumn(name = "dpt_id"),     // FK for owning side
        inverseJoinColumns = @JoinColumn(name = "doctor_id")  // FK for inverse side
    )
    private Set<Doctor> doctors = new HashSet<>();      // MUST initialize!
}
```

> **Always initialize the collection** (`new HashSet<>()`) — if Hibernate tries to populate a null Set, you get `NullPointerException`.

### 6.3 Inverse Side: Doctor (optional, for bidirectional)

```java
// Doctor.java (Inverse Side)
@Entity
public class Doctor {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String specialization;

    @Column(unique = true, length = 100)
    private String email;

    @ManyToMany(mappedBy = "doctors")    // "doctors" = field in Department
    private Set<Department> departments = new HashSet<>();
}
```

### 6.4 Two Relationships Between Same Entities

Department has **two** different relationships with Doctor:

```java
@OneToOne
private Doctor headDoctor;          // relationship 1: department head

@ManyToMany
private Set<Doctor> doctors = ...;  // relationship 2: all department doctors
```

This is perfectly valid in JPA. Two different relationships → two different columns/tables.

---

## 7. Summary — All Mapping Types

| Mapping | Annotation | Join Column/Table | Example |
|---|---|---|---|
| One-to-One | `@OneToOne` | FK column on owning side | Patient → Insurance |
| Many-to-One | `@ManyToOne` | FK column on owning side | Appointment → Patient |
| One-to-Many | `@OneToMany` | No column (mappedBy) or join table | Patient → Appointments |
| Many-to-Many | `@ManyToMany` | Join table (separate table) | Department ↔ Doctor |

---

## 8. Mapping Rules Cheat Sheet

| Rule | Details |
|---|---|
| Owning side has `@JoinColumn` | Defines the FK column |
| Inverse side has `mappedBy` | Points to field name in owning entity |
| Only owning side controls FK updates | Saving inverse side does not update FK |
| OneToOne always adds UNIQUE to FK | Prevents duplicate associations |
| OneToMany without mappedBy → join table | Almost always wrong for 1:N |
| ManyToMany always creates join table | Contains PK pair of both entities |
| Initialize collections | `= new HashSet<>()` or `= new ArrayList<>()` |

---

## 9. JPA Buddy Plugin (IntelliJ)

JPA Buddy is an IntelliJ plugin that speeds up JPA development:

**Install:** IntelliJ Preferences → Plugins → Marketplace → search "JPA Buddy"

**Features used:**
- Generate Spring Data repository interface automatically from an entity
- IntelliJ reads the entity fields and types and creates:

```java
// Auto-generated for Appointment entity:
public interface AppointmentRepository extends JpaRepository<Appointment, Long> {
}
```

**To use:** Open entity file → JPA Buddy panel appears → Spring Data Repository → choose parent (JpaRepository) → choose package → OK.

---

## 10. Repositories — One per Entity

```java
public interface PatientRepository extends JpaRepository<Patient, Long> {}
public interface InsuranceRepository extends JpaRepository<Insurance, Long> {}
public interface AppointmentRepository extends JpaRepository<Appointment, Long> {}
public interface DoctorRepository extends JpaRepository<Doctor, Long> {}
public interface DepartmentRepository extends JpaRepository<Department, Long> {}
```

---

## 11. Entity Lifecycle with Relationships

When you save an entity that has relationships, only the owning side fields are persisted to the FK column.

```java
// Create and save an appointment (owning side)
Appointment appt = new Appointment();
appt.setReason("Fever checkup");
appt.setPatient(patient);    // sets the FK patient_id
appt.setDoctor(doctor);      // sets the FK doctor_id
appointmentRepository.save(appt);  // inserts with both FKs ✅
```

---

## 12. Summary Table

| Concept | What It Means |
|---|---|
| `@OneToOne` | One-to-one relationship; adds UNIQUE FK column |
| `@OneToMany` | Parent side; no FK column unless no `mappedBy` |
| `@ManyToOne` | Child side; has the FK column (owning side) |
| `@ManyToMany` | Creates a join table; both entities are FKs+PKs |
| `@JoinColumn` | Customizes FK column name, nullable, uniqueness |
| `@JoinTable` | Customizes join table name and column names (ManyToMany) |
| `mappedBy` | Declares inverse side; value = field name on owning side |
| Owning Side | Controls FK — must be updated to persist relationship |
| Inverse Side | Read-only mapping; changes not persisted |
| ERD | Diagram showing entities, cardinality, and participation |
