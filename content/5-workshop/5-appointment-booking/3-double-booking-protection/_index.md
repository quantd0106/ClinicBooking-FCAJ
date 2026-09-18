---
title: "5.5.3. Double-booking Protection"
weight: 3
chapter: false
---

# 5.5.3. Double-booking Protection

## 1. Understand the race

An availability response is a read, not a reservation. Without transactional protection, two patients could both see 09:00 as free and both create an appointment:

```text
Patient A checks 09:00 → free
Patient B checks 09:00 → free
A writes an appointment
B writes an appointment
→ two active appointments for the same slot
```

The backend must recheck at booking time. The conflict identity is the same doctor, clinic date, and start time, with status PENDING or CONFIRMED.

## 2. Follow the real transaction

`AppointmentPersistenceService.bookAppointment()` performs the following in one Prisma/MySQL transaction:

1. Call `lockBookingSchedule()` for the selected schedule.
2. Validate the joined doctor's and doctor's user account's active flags.
3. Require the schedule's doctor and date to match the request and its status to be OPEN.
4. Require the start to lie inside the window, the computed end to fit, and the start offset to align with `slotDurationMinutes`.
5. Require the clinic-to-UTC start instant to be strictly in the future.
6. Query active appointments for that doctor/date/start time; throw `SlotAlreadyBookedError` if found.
7. Upsert the patient profile using the authenticated user ID.
8. Create the PENDING appointment and its nested initial history.
9. Commit only if every step succeeds.

The end time is calculated from the locked schedule's slot duration. The client cannot substitute it.

The following exact SQL clauses are extracted from `lockBookingSchedule()`; the selected columns and surrounding Prisma call are omitted:

```sql
FROM doctor_schedules
INNER JOIN doctors ON doctors.id = doctor_schedules.doctor_id
INNER JOIN users ON users.id = doctors.user_id
WHERE doctor_schedules.id = ${scheduleId}
FOR UPDATE
```

`${scheduleId}` is a Prisma SQL-template expression. It is not a literal resource ID or a standalone SQL command.

The schedule locking read serializes competing attempts for the same scheduling context. After the first transaction commits, the waiting attempt continues and checks the active booking before writing.

## 3. Check blocking statuses using a current read

The active statuses are defined exactly as follows:

```typescript
export const ACTIVE_APPOINTMENT_STATUSES: AppointmentStatus[] = [
  AppointmentStatus.PENDING,
  AppointmentStatus.CONFIRMED,
];
```

`isSlotActivelyBooked()` filters by doctor ID, appointment date, start time, and these statuses. It uses explicit MySQL `DATE(...)` and `TIME(...)` conversion. When called inside the booking transaction, it adds `FOR UPDATE`, so the booking path uses a locking/current read.

CANCELLED and COMPLETED are excluded from the active query. Cancelled rows and their audit history remain available; removing them is not required for rebooking.

The Prisma schema defines an ordinary index on doctor/date/start time, **not a UNIQUE constraint** on that slot. Transactional conflict checks enforce the active-booking rule while allowing a cancelled slot to be booked again.

## 4. Return the exact conflict

This excerpt is copied from `AppointmentsService.mapPersistenceError()`; unrelated branches are omitted:

```typescript
if (error instanceof SlotAlreadyBookedError) {
  throw new ConflictException({ code: ApiErrorCode.SlotAlreadyBooked, message: error.message });
}
```

The global exception filter renders:

```json
{
  "statusCode": 409,
  "code": "SLOT_ALREADY_BOOKED",
  "message": "The selected appointment slot is no longer available."
}
```

An exception prevents the transaction from committing partial appointment/history changes.

## 5. Run the safe local workshop scenario

Use your own private local PATIENT accounts and a future OPEN schedule with an aligned 09:00 slot. Follow the request DTO from [5.5.1](../1-create-appointment/).

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Log in as Patient A; keep the token private. | Ready to authenticate the request. |
| 2 | POST the selected doctor/schedule/date/09:00 with `<patient-a-token>`. | 201 Created, PENDING appointment and one initial history entry. |
| 3 | Log in as Patient B. | A separate authenticated PATIENT identity. |
| 4 | POST the same doctor/schedule/date/09:00 with `<patient-b-token>`. | 409 `SLOT_ALREADY_BOOKED`. |
| 5 | Inspect matching appointments in the authorized local database. | Exactly one PENDING/CONFIRMED appointment for that doctor/date/start. |
| 6 | Patient A cancels their original `<appointment-id>`. | 200, CANCELLED and new cancellation history. |
| 7 | Patient B retries the same booking while it remains valid and future. | 201; the cancelled appointment no longer blocks the slot. |

For a concurrent variation, send both booking attempts together. Either patient may win; expect one successful booking and one 409, not a guaranteed winner.

The inspected M4 integration test uses two concurrent `appointments.create()` calls with `Promise.allSettled()` at 09:30. It asserts one success, one 409 `SLOT_ALREADY_BOOKED`, one active appointment, and one initial history row. The 09:00 walkthrough above uses the same rule with another aligned slot.

{{< notice info >}}
Availability can change after discovery. Booking must validate and recheck inside the transaction. Cancellation frees an active slot only if the remaining doctor, schedule, and future-start conditions still permit booking.
{{< /notice >}}

> 📷 Screenshot to add: successful booking, duplicate-booking conflict, and the local database showing only one active appointment. Hide all real tokens and private identifiers.

