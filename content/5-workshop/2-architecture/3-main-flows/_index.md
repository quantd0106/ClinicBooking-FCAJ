---
title: "5.2.3. Main System Flows"
weight: 3
chapter: false
---

# 5.2.3. Main System Flows

## 1. Authentication

```text
Client
→ POST /api/v1/auth/login
→ Amazon API Gateway
→ AWS Lambda
→ NestJS Auth
→ JWT returned to Client
```

The authentication flow identifies the user and returns a JWT. Protected requests use JWT authentication and RBAC to enforce PATIENT, DOCTOR, and ADMIN permissions.

Public registration creates PATIENT accounts only. A public client cannot select a role or register itself as DOCTOR or ADMIN.

## 2. Doctor discovery and available slots

```text
Patient
→ GET specialties / doctors
→ Select doctor
→ GET /doctors/{id}/available-slots?date=YYYY-MM-DD
→ Backend combines doctor schedule and active appointments
→ Available slots returned
```

The doctor's working schedule and slot configuration define the possible appointment slots. Existing PENDING and CONFIRMED appointments remove occupied slots from the available results. CANCELLED appointments do not block rebooking.

An availability response is not a reservation: another request may book a slot before the patient submits a booking. The booking transaction therefore checks availability again.

## 3. Appointment booking

```text
Patient
→ POST /appointments
→ Validate identity and role
→ Validate schedule
→ Open MySQL transaction
→ Lock the relevant schedule
→ Check active appointments for the same doctor/date/time
→ Create PENDING appointment
→ Create status history
→ Commit
```

PENDING and CONFIRMED are the active appointment statuses. Transaction and locking logic protect against concurrent bookings for the same doctor, date, and time.

If an active booking already occupies the slot:

```text
HTTP 409
SLOT_ALREADY_BOOKED
```

The backend rejects the duplicate booking rather than creating another active appointment for the occupied slot.

## 4. Cancellation and slot release

```text
Patient cancels appointment
→ Status becomes CANCELLED
→ Status history is recorded
→ Cancelled appointment no longer blocks the slot
→ Slot becomes available for booking again
```

Cancellation follows the patient's appointment permissions. PENDING and CONFIRMED appointments may transition to CANCELLED. The cancelled appointment remains part of the appointment lifecycle and status history, while active-booking checks allow the slot to be booked again.

## 5. Doctor status transitions

```text
Doctor updates a relevant appointment
→ PENDING → CONFIRMED → COMPLETED
→ Each status change is recorded in appointment_status_history
```

Doctors view relevant appointments and update their statuses within system permissions. This flow follows the allowed lifecycle; it does not introduce additional transitions or unconfirmed administrative behavior.
