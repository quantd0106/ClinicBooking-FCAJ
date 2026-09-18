---
title: "5.5.1. Create Appointment"
weight: 1
chapter: false
---

# 5.5.1. Create Appointment

## 1. Identify the endpoint and patient identity

`POST /api/v1/appointments` requires a Bearer JWT and the PATIENT role. The controller uses `@Roles(Role.PATIENT)`, and `AppointmentsService.create()` checks the role again.

The verified request DTO contains exactly four required fields:

| Field | API validation |
| --- | --- |
| `doctorId` | Doctor UUID. |
| `scheduleId` | Working schedule UUID. |
| `appointmentDate` | Clinic date in `YYYY-MM-DD`; real-date validation also occurs through the time service. |
| `startTime` | 24-hour clinic time in `HH:mm`. |

There is no client-supplied `patientId`, `patientUserId`, `endTime`, or `status` field. The global validation Pipe rejects fields outside the DTO. The server derives the patient identity from the authenticated actor, calculates the end time from the schedule duration, and creates status PENDING.

This exact call excerpt from `src/appointments/appointments.service.ts` omits the role check, surrounding error handling, and response mapping:

```typescript
const appointment = await this.persistence.bookAppointment({
  patientUserId: actor.id,
  doctorId: dto.doctorId,
  scheduleId: dto.scheduleId,
  appointmentDate: dto.appointmentDate,
  startTime: dto.startTime,
  slotInstant: this.clinicTime.toUtcInstant(dto.appointmentDate, dto.startTime),
});
```

## 2. Prepare a safe request

Obtain the doctor ID from discovery and the schedule ID from the authorized schedule creation response. Select an aligned, future start from that OPEN schedule.

```http
POST /api/v1/appointments
Authorization: Bearer <patient-token>
```

```json
{
  "doctorId": "<doctor-id>",
  "scheduleId": "<schedule-id>",
  "appointmentDate": "2036-01-15",
  "startTime": "09:00"
}
```

Replace identifiers with valid UUIDs in your private local test. Use a future date matching the actual schedule; the illustrative date follows the existing unit-test fixture.

## 3. Follow the verified booking flow

```text
PATIENT JWT and DTO validation
→ authenticated actor ID and clinic-to-UTC slot instant
→ MySQL transaction
→ lock schedule and validate doctor/account, date, range, alignment, future start
→ check active appointment conflicts
→ upsert patient profile using the authenticated user ID
→ create PENDING appointment with initial history
→ commit and return detail response
```

The patient profile is resolved with `patient.upsert()` inside the transaction, after the schedule and conflict checks. This order follows the real persistence code. A missing profile can be created as an identity-only profile; no personal details are invented.

## 4. Interpret the successful response

The controller declares 201 Created with `AppointmentDetailResponseDto`. This sanitized example follows the stored appointment fixture and response mapping in the service unit test; UUIDs are replaced by placeholders. It is not a newly executed response.

```json
{
  "id": "<appointment-id>",
  "patientId": "<patient-id>",
  "doctorId": "<doctor-id>",
  "scheduleId": "<schedule-id>",
  "appointmentDate": "2036-01-15",
  "startTime": "09:00",
  "endTime": "09:30",
  "status": "PENDING",
  "createdAt": "2030-01-01T00:00:00.000Z",
  "updatedAt": "2030-01-01T00:00:00.000Z",
  "history": [
    {
      "id": "<history-id>",
      "oldStatus": null,
      "newStatus": "PENDING",
      "changedBy": "<patient-user-id>",
      "changedAt": "2030-01-01T00:00:00.000Z",
      "reason": null
    }
  ]
}
```

The appointment's `patientId` identifies the patient profile, whereas history `changedBy` identifies the actor's user account. Calendar fields retain clinic date/time; audit timestamps are ISO-8601 UTC strings.

| Failure | Actual response |
| --- | --- |
| Malformed DTO or invalid calendar date | 400 `VALIDATION_ERROR`. |
| Missing schedule in the booking join | 404 `NOT_FOUND`. |
| Inactive doctor or linked doctor account | 400 `DOCTOR_INACTIVE`. |
| Wrong doctor/date for the selected schedule, CLOSED schedule, misaligned/out-of-range/past start | 400 `INVALID_APPOINTMENT_SLOT`. |
| Existing active booking for the slot | 409 `SLOT_ALREADY_BOOKED`. |

Missing/invalid authentication returns 401; a non-PATIENT role returns 403.

## 5. View appointments with server-side visibility

| Endpoint | Verified visibility | Response |
| --- | --- | --- |
| `GET /api/v1/appointments/me` | PATIENT: own appointments; DOCTOR: appointments assigned to their doctor profile; ADMIN: all appointments. | 200 paginated list. |
| `GET /api/v1/appointments/:id` | PATIENT: own appointment; DOCTOR: assigned appointment; ADMIN: any appointment. | 200 detail including history. |

The list accepts optional `status` and clinic `date`, plus `page` (default 1) and `pageSize` (default 20, maximum 100). It orders by appointment date descending, start time descending, then ID ascending. The envelope has `items`, `page`, `pageSize`, `totalItems`, and `totalPages`; list items omit history.

For list access, a PATIENT/DOCTOR account without its required profile returns 403 `PROFILE_REQUIRED`. A missing detail record returns 404; an existing record outside the user's ownership/assignment returns 403 `OWNERSHIP_REQUIRED`.

> 📷 Screenshot to add: successful appointment creation, HTTP 201, with real tokens and personal identifiers hidden.

