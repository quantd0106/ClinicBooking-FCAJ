---
title: "5.5.2. Appointment Lifecycle"
weight: 2
chapter: false
---

# 5.5.2. Appointment Lifecycle

## 1. Follow the allowed states

An appointment starts as PENDING. The verified `assertTransition()` implementation allows only the following edges:

{{< mermaid >}}
graph TD
  PENDING --> CONFIRMED
  CONFIRMED --> COMPLETED
  PENDING --> CANCELLED
  CONFIRMED --> CANCELLED
{{< /mermaid >}}

COMPLETED and CANCELLED are terminal in this implementation. Repeating a status or completing directly from PENDING is not allowed.

## 2. Apply roles and ownership

All appointment endpoints require JWT authentication. The transition endpoints return 200 with the appointment detail and history when successful.

| Endpoint | Permitted actors | Target |
| --- | --- | --- |
| `PATCH /api/v1/appointments/:id/cancel` | Owning PATIENT, assigned DOCTOR, or ADMIN overriding ownership. | CANCELLED from PENDING or CONFIRMED. |
| `PATCH /api/v1/appointments/:id/status` | Assigned DOCTOR, or ADMIN overriding assignment. | CONFIRMED from PENDING; COMPLETED from CONFIRMED. |

ADMIN overrides ownership/assignment, not lifecycle validity. A PATIENT cannot use the status endpoint, and an unrelated doctor cannot transition another doctor's assigned appointment.

## 3. Cancel with an optional reason

```http
PATCH /api/v1/appointments/<appointment-id>/cancel
Authorization: Bearer <patient-token>
```

```json
{
  "reason": "Schedule conflict"
}
```

`CancelAppointmentDto` permits an optional string `reason` of up to 500 characters; a body is not required. Cancellation writes CANCELLED history with the actor and supplied reason.

CANCELLED no longer participates in active-slot conflict checks. Rebooking can succeed if the schedule remains OPEN, the doctor/account are active, and the slot is aligned and still in the future.

## 4. Confirm, then complete

Use the assigned doctor's private token or an ADMIN token with the status endpoint:

```http
PATCH /api/v1/appointments/<appointment-id>/status
Authorization: Bearer <doctor-token>
```

First, for a PENDING appointment:

```json
{
  "status": "CONFIRMED"
}
```

Then, for a CONFIRMED appointment:

```json
{
  "status": "COMPLETED"
}
```

`UpdateAppointmentStatusDto` accepts only CONFIRMED or COMPLETED. It contains no reason field. Use the dedicated cancel endpoint for cancellation.

## 5. Reject invalid or conflicting changes

This exact method is copied from `src/database/appointment-persistence.service.ts`; the surrounding class is omitted:

```typescript
private assertTransition(oldStatus: AppointmentStatus, newStatus: AppointmentStatus): void {
  const allowed =
    (oldStatus === AppointmentStatus.PENDING &&
      (newStatus === AppointmentStatus.CONFIRMED || newStatus === AppointmentStatus.CANCELLED)) ||
    (oldStatus === AppointmentStatus.CONFIRMED &&
      (newStatus === AppointmentStatus.COMPLETED || newStatus === AppointmentStatus.CANCELLED));
  if (!allowed) {
    throw new InvalidAppointmentTransitionError();
  }
}
```

The persistence layer first reads the observed status and validates the transition. Inside a transaction it locks the appointment with `FOR UPDATE`, checks that the status still matches the observation, checks actor permissions, validates the transition again, updates the status, and inserts history atomically.

| Failure | Response |
| --- | --- |
| Unsupported DTO target, such as PENDING/CANCELLED on the status endpoint | 400 `VALIDATION_ERROR`. |
| DTO-valid target not allowed from the current state, or invalid cancellation | 400 `INVALID_APPOINTMENT_TRANSITION`. |
| Current status changed between observation and the locked read | 409 `APPOINTMENT_CHANGED`. |
| Actor lacks ownership/assignment for an otherwise permitted action | 403 `OWNERSHIP_REQUIRED`. |
| Appointment not found | 404 `NOT_FOUND`. |

Authentication/role guard failures remain 401/403. Invalid transition checks are independent of role permission; ADMIN also receives an error for unsupported state changes.

> 📷 Screenshot to add: a permitted appointment transition and its new history entry, with private identifiers hidden.

