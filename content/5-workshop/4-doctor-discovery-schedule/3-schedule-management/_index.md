---
title: "5.4.3. Doctor Schedule Management"
weight: 3
chapter: false
---

# 5.4.3. Doctor Schedule Management

## 1. Identify endpoints and ownership

Schedules define when a doctor works and how that window is divided into appointment slots. The implementation is in `src/schedules/schedules.controller.ts`, `schedules.service.ts`, `dto/schedule.dto.ts`, and `src/database/schedule-persistence.service.ts`.

| Method | Endpoint | Success | Access |
| --- | --- | --- | --- |
| POST | `/api/v1/doctors/:id/schedules` | 201; schedule response. | Bearer JWT, DOCTOR, own doctor profile. |
| PATCH | `/api/v1/schedules/:id` | 200; schedule response. | Bearer JWT, DOCTOR, own schedule. |
| DELETE | `/api/v1/schedules/:id` | 204; no body. | Bearer JWT, DOCTOR, own schedule. |

All three handlers specify `@Roles(Role.DOCTOR)`. They do not grant ADMIN access. The service compares the actor's user ID with the schedule doctor's `userId` using `OwnershipService.assertOwner()`, without enabling its optional ADMIN override.

Missing/invalid authentication returns 401. A disallowed role returns 403 `FORBIDDEN`; a doctor managing another doctor's schedule receives 403 `OWNERSHIP_REQUIRED`.

## 2. Prepare a valid create request

The verified `CreateScheduleDto` fields are:

| Field | Validation and meaning |
| --- | --- |
| `workDate` | Required date in `YYYY-MM-DD`; the time service also checks the real calendar date. |
| `startTime` | Required 24-hour `HH:mm` start. |
| `endTime` | Required 24-hour `HH:mm` end; must be after start. |
| `slotDurationMinutes` | Required integer, 1–1440 at the API boundary; must fit inside the window. |
| `status` | Optional `OPEN` or `CLOSED`; persistence defaults to OPEN. |

Use a doctor UUID from your local data and that doctor's private token:

```http
POST /api/v1/doctors/<doctor-id>/schedules
Authorization: Bearer <token>
```

```json
{
  "workDate": "2030-01-15",
  "startTime": "09:00",
  "endTime": "12:00",
  "slotDurationMinutes": 30,
  "status": "OPEN"
}
```

Choose a future date when following the example. The service requires the start, interpreted in the clinic timezone, to be strictly after the current time.

The schedule response contains `id`, `doctorId`, `workDate`, `startTime`, `endTime`, `slotDurationMinutes`, and `status`. Request date/time strings are converted through `ClinicTimeService` to the schema's MySQL date/time fields; responses format the calendar date and time back into strings.

## 3. Prevent overlapping working windows

For one doctor on one date, an existing 09:00–12:00 window overlaps a new 10:00–13:00 window. Rejecting it avoids two working windows producing conflicting slot definitions.

`SchedulePersistenceService.createSchedule()` normalizes the input, opens a Prisma transaction, locks the doctor row with `SELECT ... FOR UPDATE`, checks overlap with a locking/current read, and creates the schedule only when clear.

This exact SQL excerpt from `assertNoOverlap()` omits its surrounding query and optional exclusion of the schedule being updated:

```sql
WHERE doctor_id = ${input.doctorId}
  AND work_date = DATE(${input.workDate})
  AND start_time < TIME(${input.endTime})
  AND end_time > TIME(${input.startTime})
```

These are Prisma SQL-template expressions, not values to paste into a SQL console. Both comparisons are strict, so adjacent windows can meet at a boundary. The overlap query checks all schedule statuses, including CLOSED.

The locking/current read avoids missing a concurrently committed schedule through an older transaction snapshot. Update also locks the doctor and schedule rows and excludes the current schedule from its overlap check.

A conflict returns:

```json
{
  "statusCode": 409,
  "code": "SCHEDULE_OVERLAP",
  "message": "The doctor already has an overlapping schedule on this date."
}
```

The supplied historical September 10, 2026 example—09:00–12:00 with 30-minute slots returning 201, followed by an overlapping 10:00–13:00 request returning 409—is consistent with this rule. It is not a new test run. Repeating creation after that date requires a future date because of the future-start validation.

## 4. Update or delete without damaging bookings

`UpdateScheduleDto` is a partial version of the create DTO. Supply at least one field. The service merges the submitted fields with the existing schedule and validates the resulting future start.

Before update or deletion, persistence checks for active appointments. If present, it returns 409 `SCHEDULE_HAS_ACTIVE_BOOKINGS`. The update check applies to any schedule update, not only a time change.

Deletion is transactional:

- Active appointments: reject with 409.
- Appointment history remains but none are active: set the schedule to CLOSED.
- No appointments refer to the schedule: physically delete it.

Malformed UUIDs, invalid windows/durations, an empty PATCH body, or a non-future start return 400. Missing doctors or schedules return 404 `NOT_FOUND`.

{{< notice info >}}
Schedule overlap prevention keeps one doctor's working windows from overlapping. Appointment double-booking prevention keeps two active appointments from occupying the same doctor/date/start-time slot. These are separate rules; appointment concurrency protection is covered in section 5.5.
{{< /notice >}}

## 5. Check in local Swagger

With your private DOCTOR account, create a future schedule, try an overlapping window, and check ownership by targeting another doctor's schedule. Relevant M3 tests cover role restrictions, ownership, overlap, and protection of schedules with active bookings.

> 📷 Screenshot to add: successful schedule creation and the 409 SCHEDULE_OVERLAP response, with tokens and personal identifiers hidden.

