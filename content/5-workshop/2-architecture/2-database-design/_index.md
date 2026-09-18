---
title: "5.2.2. Database Design"
weight: 2
chapter: false
---

# 5.2.2. Database Design

## Data-access approach

The backend accesses MySQL through Prisma ORM. The core data model separates authentication identities, patient and doctor profiles, specialties, working schedules, appointments, and appointment status history.

## Core entities and relationships

| Entity | Purpose | Main relationship |
| --- | --- | --- |
| users | Authentication identity with the PATIENT, DOCTOR, or ADMIN role. | Patient and doctor profiles are linked to their user identity. |
| patients | Patient profile. | Linked to users and associated with the patient's appointments. |
| doctors | Doctor profile. | Linked to users; associated with specialties, working schedules, and appointments. |
| specialties | Medical specialty. | Related to doctors through doctor_specialties. |
| doctor_specialties | Mapping between doctors and specialties. | Supports the many-to-many relationship between doctors and specialties. |
| doctor_schedules | Doctor working schedule and slot configuration. | Associated with a doctor and used to determine available appointment slots. |
| appointments | Patient booking with a doctor, date, time, and status. | Associated with a patient and doctor; has recorded status changes. |
| appointment_status_history | Audit trail for every appointment status transition. | Associated with the appointment whose status changes. |

## Logical relationship diagram

This logical ERD shows how user identities, profiles, specialties, working schedules, appointments, and status history relate to one another.

{{< mermaid >}}
graph LR
  Users["users"] -->|patient profile| Patients["patients"]
  Users -->|doctor profile| Doctors["doctors"]
  Doctors -->|working schedule| Schedules["doctor_schedules"]
  Doctors -->|mapping| DoctorSpecialties["doctor_specialties"]
  Specialties["specialties"] -->|mapping| DoctorSpecialties
  Patients -->|patient booking| Appointments["appointments"]
  Doctors -->|doctor booking| Appointments
  Appointments -->|status transitions| History["appointment_status_history"]
{{< /mermaid >}}

## Appointment statuses

| Status | Meaning |
| --- | --- |
| PENDING | A booking has been created and is awaiting confirmation. It is active and blocks the slot. |
| CONFIRMED | The appointment has been confirmed. It remains active and blocks the slot. |
| COMPLETED | The confirmed appointment has been completed. |
| CANCELLED | The appointment has been cancelled and no longer blocks rebooking. |

The allowed lifecycle is:

```text
PENDING → CONFIRMED → COMPLETED
PENDING → CANCELLED
CONFIRMED → CANCELLED
```

Every appointment status change is recorded in appointment_status_history.

## Booking consistency

Available-slot calculation combines the doctor's working schedule and slot configuration with active appointments. PENDING and CONFIRMED are the active statuses checked for conflicts involving the same doctor, date, and time.

Booking uses a MySQL transaction and locking logic. A duplicate active booking returns HTTP 409 with SLOT_ALREADY_BOOKED. CANCELLED appointments are excluded from active-booking checks and therefore do not block the same slot from being booked again.
