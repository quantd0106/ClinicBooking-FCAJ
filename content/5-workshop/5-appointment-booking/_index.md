---
title: "5.5. Appointment Booking Core"
weight: 5
chapter: false
---

# 5.5. Appointment Booking Core

This stage explains the system's central business logic: creating and viewing appointments, cancellation, doctor status transitions, protection against concurrent booking, and status audit history.

Follow the existing backend through its appointment controller, service, and persistence layer. Use the local Swagger UI from 5.3 and the doctor schedules from 5.4. All examples use placeholders instead of real tokens or identifiers.

## Sections

1. [5.5.1. Create Appointment](1-create-appointment/)
2. [5.5.2. Appointment Lifecycle](2-appointment-lifecycle/)
3. [5.5.3. Double-booking Protection](3-double-booking-protection/)
4. [5.5.4. Appointment Status History](4-status-history/)

## Verified business rules

- PENDING and CONFIRMED appointments block a slot.
- CANCELLED appointments do not block rebooking; the slot must still satisfy the schedule and future-start checks.
- Invalid lifecycle transitions are rejected, including for ADMIN.
- Booking creation and its initial status history succeed atomically.
- Role and ownership rules are enforced by the server.

