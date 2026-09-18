---
title: "5.4.4. Available Slot Calculation"
weight: 4
chapter: false
---

# 5.4.4. Available Slot Calculation

## 1. Request slots for a clinic date

The public endpoint is `GET /api/v1/doctors/:id/available-slots?date=YYYY-MM-DD`. It is implemented by `SchedulesController` and `SchedulesService.availableSlots()`; no JWT is required.

```http
GET /api/v1/doctors/<doctor-id>/available-slots?date=2026-09-10
```

Replace `<doctor-id>` with a valid doctor UUID from discovery. This is the historical date supplied for the workshop. When it is already in the past, the algorithm removes all past starts; use a future clinic date with an OPEN schedule to inspect upcoming availability.

`AvailableSlotsQueryDto` requires `date` in `YYYY-MM-DD`. The time service checks the real calendar date as well as the format.

## 2. Follow the actual calculation

1. Look up the doctor, requiring both the doctor and linked user to be active.
2. Parse the requested clinic date.
3. Load that doctor's OPEN schedules for the date, ordered by start time then ID.
4. In parallel, load appointments for that doctor/date whose statuses are PENDING or CONFIRMED.
5. Build a set of occupied start times.
6. Split each working window using `slotDurationMinutes`, keeping only complete slots whose end fits inside the window.
7. Convert each candidate's clinic date/time to a UTC instant. Skip starts at or before the current time and starts already occupied.
8. Return the remaining slots in the response envelope.

The shared `ACTIVE_APPOINTMENT_STATUSES` constant contains exactly PENDING and CONFIRMED. CANCELLED and COMPLETED appointments are not included in the occupied-start query; CANCELLED therefore does not prevent rebooking.

This exact excerpt from `SchedulesService.availableSlots()` omits the generation loop around it:

```typescript
const startAt = this.clinicTime.toUtcInstant(date, displayStart);
if (startAt <= now || bookedStarts.has(`${displayStart}:00`)) {
  continue;
}
generated.push({
  startAt: startAt.toISOString(),
  endAt: this.clinicTime.toUtcInstant(date, displayEnd).toISOString(),
  displayStart,
  displayEnd,
  available: true as const,
});
```

Availability is calculated when queried. The later appointment operation must still enforce its own booking rules; this read response is not a reservation.

## 3. Interpret the response and timezone

The verified envelope contains `doctorId`, `date`, `timezone`, and `slots`. Every slot contains `startAt`, `endAt`, `displayStart`, `displayEnd`, and `available: true`.

`ClinicTimeService` uses the configured clinic timezone, `Asia/Ho_Chi_Minh`, to convert local calendar values into UTC instants. `startAt` and `endAt` are ISO-8601 UTC strings; display times remain `HH:mm` in the clinic timezone.

For a concrete source-backed example, the Schedules service unit test fixes the clock at `2030-01-15T02:15:00.000Z` (09:15 clinic time). Its OPEN 09:00–11:00 schedule uses 30-minute slots, and 09:30 is occupied. The result excludes the past 09:00 start and occupied 09:30 start:

```json
{
  "doctorId": "<doctor-id>",
  "date": "2030-01-15",
  "timezone": "Asia/Ho_Chi_Minh",
  "slots": [
    {
      "startAt": "2030-01-15T03:00:00.000Z",
      "endAt": "2030-01-15T03:30:00.000Z",
      "displayStart": "10:00",
      "displayEnd": "10:30",
      "available": true
    },
    {
      "startAt": "2030-01-15T03:30:00.000Z",
      "endAt": "2030-01-15T04:00:00.000Z",
      "displayStart": "10:30",
      "displayEnd": "11:00",
      "available": true
    }
  ]
}
```

This is the test's expected response with its dummy doctor ID replaced by a placeholder, not a fresh backend response.

## 4. Distinguish errors from empty availability

| Condition | Actual outcome |
| --- | --- |
| Malformed doctor UUID | 400 `VALIDATION_ERROR` at the API boundary. |
| Well-formed UUID with no doctor, inactive doctor, or inactive linked user | 404 `NOT_FOUND`. |
| Missing/malformed date, or impossible calendar date | 400 `VALIDATION_ERROR` when date validation is reached. |
| No OPEN schedules on that date | 200 with `slots: []`. |
| All generated slots are occupied or in the past | 200 with `slots: []`. |

A wrong but well-formed doctor ID causing 404, followed by a valid active doctor returning 200, is consistent with the historical manual check. An empty slot array is not a missing-doctor error.

## 5. Check the calculation locally

In Swagger, select an active doctor and future date with a known OPEN schedule. Compare the slot count and display times with that schedule. The inspected unit test checks past-slot exclusion, active appointments, and UTC output; the M3 persistence integration test also covers CANCELLED not blocking a slot.

> 📷 Screenshot to add: available-slots response showing UTC timestamps and clinic display times, with private identifiers hidden.

