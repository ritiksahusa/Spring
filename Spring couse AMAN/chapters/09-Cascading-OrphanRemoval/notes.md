# Chapter 13 — Cascading, OrphanRemoval and Mapping Operations
**Timestamps:** `5:45:42 → 6:36:30`
**Topics:** CascadeType, orphanRemoval, dirty checking, @Transactional, FetchType, bidirectional consistency, save/update/delete operations on related entities

---

## 1. Two Different Relationship Domains

JPA has two distinct concepts that beginners often confuse:

| Domain | Concept | Key Annotation |
|---|---|---|
| **JPA/Mapping Domain** | Owning Side vs Inverse Side | `@JoinColumn` = owning, `mappedBy` = inverse |
| **Data Domain** | Parent vs Child | Who can exist without the other? |

**Parent** = entity that can exist independently (Patient can exist without appointments)
**Child** = entity that cannot exist meaningfully without parent (Appointment without a Patient = meaningless)

In `Patient ↔ Appointment`:
- JPA domain: Appointment = Owning Side (has `@JoinColumn`)
- Data domain: Patient = Parent, Appointment = Child

These are DIFFERENT concepts — knowing both is essential for interviews.

---

## 2. Cascading in JPA

Cascading means: **When an operation is performed on the parent entity, automatically perform the same operation on the child entity.**

### 2.1 CascadeType Options

```java
@OneToOne(cascade = CascadeType.PERSIST)     // array supported too
@OneToMany(cascade = {CascadeType.PERSIST, CascadeType.REMOVE})
@OneToMany(cascade = CascadeType.ALL)        // includes all types
```

| CascadeType | Triggered When... | Effect on Child |
|---|---|---|
| `PERSIST` | Parent is saved for first time | Child is also inserted |
| `MERGE` | Parent is updated (merged) | Child is also updated |
| `REMOVE` | Parent is deleted | All children are also deleted |
| `REFRESH` | `entityManager.refresh(parent)` is called | Child is also refreshed from DB |
| `DETACH` | Parent is detached from PersistenceContext | Child is also detached |
| `ALL` | Any of the above | Applies all cascade types |

### 2.2 PERSIST vs MERGE

```java
@OneToOne(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private Insurance insurance;
```

- `PERSIST` — when you call `save()` on a parent for the FIRST TIME, the new child is also inserted.
- `MERGE` — when you update a parent (calling save again on an existing entity), the child is also updated/inserted.

In Spring Data JPA, `repository.save()` calls either `persist` (new) or `merge` (existing) depending on whether the entity has an ID.

### 2.3 Without Cascade — What Happens?

```java
@OneToOne   // no cascade
private Insurance insurance;
```

If you set `patient.setInsurance(newInsurance)` and save patient:
```
Error: org.hibernate.TransientPropertyValueException:
object references an unsaved transient instance - save the transient instance before flushing
```

The insurance object is in **transient state** (not yet saved). Without cascade, JPA doesn't know it should save it automatically.

### 2.4 With CascadeType.MERGE — The Fix

```java
@OneToOne(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private Insurance insurance;
```

Now when you set `patient.setInsurance(insurance)` inside a `@Transactional` method:
1. At transaction commit (flush phase), dirty checking detects patient is dirty
2. Patient has a new Insurance that's not saved yet
3. Because `CascadeType.PERSIST/MERGE` is set, Hibernate first inserts the Insurance
4. Gets back the generated Insurance ID
5. Updates Patient with the new `insurance_id` FK
6. All without any explicit `insuranceRepository.save(insurance)` call

---

## 3. Practical: Assign Insurance to Patient

```java
@Service
@RequiredArgsConstructor
public class InsuranceService {
    private final InsuranceRepository insuranceRepository;
    private final PatientRepository patientRepository;

    @Transactional
    public Patient assignInsuranceToPatient(Insurance insurance, Long patientId) {
        Patient patient = patientRepository.findById(patientId)
            .orElseThrow(() -> new EntityNotFoundException("Patient not found: " + patientId));

        patient.setInsurance(insurance);          // marks patient as DIRTY
        insurance.setPatient(patient);             // bidirectional consistency (optional but good practice)

        return patient;                            // no explicit save needed — dirty checking handles it
    }
}
```

**Why `return patient` and no `patientRepository.save()`?**
- `@Transactional` keeps the PersistenceContext open
- `patient` is in **persistent state** (was loaded from DB)
- Setting `insurance` makes it dirty
- At transaction commit, Hibernate's dirty checking auto-issues the UPDATE
- Because `CascadeType.PERSIST/MERGE` is set on insurance, Hibernate inserts it first

---

## 4. CascadeType.REMOVE — Delete Parent, Delete Children

```java
// Patient.java
@OneToMany(mappedBy = "patient", cascade = CascadeType.REMOVE)
private List<Appointment> appointments = new ArrayList<>();
```

When you delete a patient:
```java
patientRepository.deleteById(1L);
// Hibernate generates:
// DELETE FROM appointment WHERE patient_id = 1
// DELETE FROM patient WHERE id = 1
```

All the patient's appointments are deleted first (child before parent), then the patient is deleted.

> **When to use:** When children have no meaning without the parent (e.g., Appointment without a Patient).

> **Danger on child side:** Never put `CascadeType.REMOVE` on a child entity's relationship to parent. Example: if Appointment has `@ManyToOne(cascade = CascadeType.REMOVE) private Patient patient` → deleting one Appointment would try to delete the Patient, which would also try to delete other Appointments → constraint violation or data loss.

---

## 5. OrphanRemoval

### 5.1 Definition

```java
@OneToMany(mappedBy = "patient", orphanRemoval = true)
private List<Appointment> appointments = new ArrayList<>();
```

OrphanRemoval deletes a child entity when it is **removed from its parent's collection** — even if the parent itself is NOT deleted.

### 5.2 OrphanRemoval vs CascadeType.REMOVE

| Feature | CascadeType.REMOVE | orphanRemoval = true |
|---|---|---|
| Triggers when... | Parent entity is deleted | Child is removed from parent's collection OR parent is deleted |
| Parent remains? | No — parent is deleted | Yes — parent can still exist |
| Example operation | `patientRepository.delete(patient)` | `patient.getAppointments().remove(appt)` |
| Use case | Wipe all children when parent is gone | Child has no standalone meaning |

### 5.3 How OrphanRemoval Works in Practice

```java
// Set insurance to null — patient stays, insurance is deleted
@Transactional
public Patient dissociateInsuranceFromPatient(Long patientId) {
    Patient patient = patientRepository.findById(patientId).orElseThrow(...);
    patient.setInsurance(null);    // marks insurance as orphan
    return patient;
    // at transaction commit: UPDATE patient SET insurance_id = NULL
    //                         DELETE FROM insurance WHERE id = <old_id>
}
```

For OneToMany:
```java
// Remove one appointment from list — appointment is deleted from DB
patient.getAppointments().remove(someAppointment);
// Hibernate: DELETE FROM appointment WHERE id = <someAppointment.id>
```

> **Key rule:** `orphanRemoval = true` can only be placed on `@OneToOne` or `@OneToMany`. It doesn't work on `@ManyToOne` or `@ManyToMany`.

---

## 6. Practical: Create Appointment

```java
@Service
@RequiredArgsConstructor
public class AppointmentService {
    private final AppointmentRepository appointmentRepository;
    private final DoctorRepository doctorRepository;
    private final PatientRepository patientRepository;

    @Transactional
    public Appointment createNewAppointment(Appointment appointment, Long doctorId, Long patientId) {
        // Guard: ensure appointment is new (no ID)
        if (appointment.getId() != null) {
            throw new IllegalArgumentException("Appointment already has an ID — use update instead");
        }

        Doctor doctor = doctorRepository.findById(doctorId).orElseThrow(...);
        Patient patient = patientRepository.findById(patientId).orElseThrow(...);

        // Appointment owns both relationships — set on owning side
        appointment.setPatient(patient);
        appointment.setDoctor(doctor);

        // Bidirectional consistency (good practice)
        patient.getAppointments().add(appointment);

        return appointmentRepository.save(appointment);  // explicit save since it's a new (transient) entity
    }
}
```

**Why explicit save here?** Because appointment is in **transient state** — it was not loaded from DB. Dirty checking only works on entities in persistent state. New entities must be explicitly saved.

---

## 7. Practical: Reassign Appointment to Another Doctor

```java
@Transactional
public Appointment reassignAppointmentToAnotherDoctor(Long appointmentId, Long newDoctorId) {
    Appointment appointment = appointmentRepository.findById(appointmentId).orElseThrow(...);
    Doctor newDoctor = doctorRepository.findById(newDoctorId).orElseThrow(...);

    appointment.setDoctor(newDoctor);    // marks appointment as DIRTY

    return appointment;    // no .save() needed — dirty checking auto-issues UPDATE
}
```

**How it works:**
1. `appointment` is now in persistent state (loaded from DB)
2. `setDoctor()` makes it dirty
3. At transaction commit, Hibernate detects the change and issues `UPDATE appointment SET doctor_id = ? WHERE id = ?`
4. No explicit `appointmentRepository.save()` needed

---

## 8. FetchType — Lazy vs Eager

### 8.1 Default Fetch Types

| Mapping | Default FetchType | Behavior |
|---|---|---|
| `@OneToOne` | `EAGER` | Insurance loaded automatically with Patient |
| `@ManyToOne` | `EAGER` | Doctor/Patient loaded automatically with Appointment |
| `@OneToMany` | `LAZY` | Appointments loaded only when explicitly accessed |
| `@ManyToMany` | `LAZY` | Departments/Doctors loaded only when explicitly accessed |

### 8.2 LazyInitializationException

```java
// PROBLEM: session closed before accessing lazy collection
Patient patient = patientRepository.findById(1L).orElseThrow();
// session closes here (no @Transactional)
patient.getAppointments();  // LazyInitializationException! No session to load from
```

**Fixes:**
1. Add `@Transactional` to the calling method — keeps session open
2. Use `@ToString.Exclude` on the lazy field — don't access it in toString
3. Change to `FetchType.EAGER` — but this can cause N+1 problems

```java
// Fix 1: @Transactional keeps session open
@Transactional
public Patient getPatientWithAppointments(Long id) {
    Patient patient = patientRepository.findById(id).orElseThrow();
    patient.getAppointments().size();   // triggers lazy load while session is open
    return patient;
}

// Fix 2: Exclude from toString
@ToString.Exclude
@OneToMany(mappedBy = "patient")
private List<Appointment> appointments = new ArrayList<>();
```

---

## 9. Bidirectional Consistency

When you have bidirectional mapping, both sides need to be kept in sync in your code (within the same session/transaction). This is NOT required for JPA to persist correctly, but it ensures the in-memory object graph is consistent.

```java
// Setting owning side (REQUIRED for DB update)
appointment.setPatient(patient);

// Setting inverse side (optional, for in-memory consistency)
patient.getAppointments().add(appointment);
```

Why maintain both sides? If you later access `patient.getAppointments()` in the same session, the list should reflect the new appointment. Without manually adding to the list, the DB will be correct but the in-memory object might be stale.

---

## 10. Summary — Cascade and OrphanRemoval Rules

| Scenario | Solution |
|---|---|
| Save parent + auto-save new child | `cascade = {PERSIST, MERGE}` on parent |
| Delete parent + auto-delete all children | `cascade = REMOVE` or `cascade = ALL` on parent |
| Remove child from collection + auto-delete from DB | `orphanRemoval = true` on parent |
| Reassign child to different parent | No cascade needed — just `setParent()` on owning side |
| Only update/create child with parent | `cascade = {PERSIST, MERGE}` |

**Golden rules:**
1. **Never** put cascade on the child side (`@ManyToOne`) — it can accidentally delete parent when child is deleted
2. **Parent dictates lifecycle** — cascade is defined on parent → child direction
3. `orphanRemoval = true` is only valid on `@OneToOne` and `@OneToMany`
4. Cascade works in both owning and inverse sides (though owning side is preferred for clarity)

---

## 11. Lombok @Builder Pattern

```java
@Entity @Builder @NoArgsConstructor @AllArgsConstructor @Getter @Setter
public class Insurance {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String policyNumber;
    private String provider;
    private LocalDate validUntil;
}

// Usage:
Insurance insurance = Insurance.builder()
    .policyNumber("HDFC_1234")
    .provider("HDFC Bank")
    .validUntil(LocalDate.of(2030, 12, 1))
    .build();
```

`@Builder` from Lombok generates a static `builder()` method and a fluent API. You don't need to remember constructor argument order.

---

## 12. Key Concepts Summary

| Concept | Description |
|---|---|
| CascadeType.PERSIST | Saves new child automatically when parent is saved |
| CascadeType.MERGE | Updates child when parent is updated |
| CascadeType.REMOVE | Deletes children when parent is deleted |
| CascadeType.ALL | Applies all cascade types |
| orphanRemoval=true | Deletes child when it's removed from parent collection (parent still exists) |
| Dirty Checking | Auto-issues UPDATE when persistent entity is modified inside @Transactional |
| Parent (Data Domain) | Entity that dictates lifecycle of child |
| Child (Data Domain) | Entity that cannot meaningfully exist without parent |
| EAGER FetchType | Load related entity immediately (default for @OneToOne, @ManyToOne) |
| LAZY FetchType | Load only when accessed (default for @OneToMany, @ManyToMany) |
| Bidirectional Consistency | Keeping both sides of a relationship in sync in Java memory |
