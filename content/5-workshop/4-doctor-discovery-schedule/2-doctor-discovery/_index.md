---
title: "5.4.2. Doctor Discovery"
weight: 2
chapter: false
---

# 5.4.2. Doctor Discovery

## 1. Identify public and administrative operations

The implementation is in `src/doctors/doctors.controller.ts`, `doctors.service.ts`, and `dto/doctor.dto.ts`.

| Method | Endpoint | Purpose | Access |
| --- | --- | --- | --- |
| GET | `/api/v1/doctors` | List/search active doctors; 200. | Public. |
| GET | `/api/v1/doctors/:id` | Retrieve an active doctor; 200. | Public. |
| PATCH | `/api/v1/doctors/:id` | Activate/deactivate a doctor; 200. | Bearer JWT, ADMIN. |

The activation operation uses PATCH on the doctor resource itself. The controller defines `@Patch(':id')`; it does not define a `/activation` suffix.

## 2. Browse and filter doctors

```text
Client
→ GET /api/v1/doctors
→ DoctorsService builds discovery query
→ Prisma loads doctors and active specialty associations
→ response
```

The verified query parameters are:

| Parameter | Behavior |
| --- | --- |
| `search` | Optional, trimmed, maximum 150 characters; matches `displayName` using `contains`. |
| `specialtyId` | Optional UUID; requires an association with that active specialty. |
| `page` | Integer, minimum 1; default 1. |
| `pageSize` | Integer, 1–100; default 20. |

```http
GET /api/v1/doctors?search=Nguyen&page=1&pageSize=20
GET /api/v1/doctors?specialtyId=<specialty-id>&page=1&pageSize=20
```

Replace `<specialty-id>` with a UUID from your local specialty response. Discovery requires both `doctor.isActive` and the linked user's `isActive`. It includes only active specialties, ordered by specialty name, and orders doctors by display name then ID.

The list envelope contains `items`, `page`, `pageSize`, `totalItems`, and `totalPages`. Each doctor contains `id`, `name` (mapped from `displayName`), `bio`, `isActive`, and `specialties`; each specialty association in the response has `id` and `name`.

## 3. Retrieve doctor detail

```http
GET /api/v1/doctors/<doctor-id>
```

Use the doctor ID from discovery, not the user account ID. This exact query excerpt comes from `DoctorsService.detail()`; the error and response mapping are omitted:

```typescript
const doctor = await this.prisma.doctor.findFirst({
  where: { id, isActive: true, user: { isActive: true } },
  include: doctorInclude,
});
```

The detail response has the same doctor fields as a list item. A missing doctor, inactive doctor, or inactive linked user returns 404 `NOT_FOUND`. A malformed route UUID returns 400. An empty list remains a successful paginated response.

## 4. Control activation as ADMIN

Activation controls whether a doctor is available for public discovery, so it is an administrative operation. The handler uses `JwtAuthGuard`, `RolesGuard`, and `@Roles(Role.ADMIN)`.

The verified request DTO contains only the required boolean `isActive`:

```http
PATCH /api/v1/doctors/<doctor-id>
Authorization: Bearer <token>
```

```json
{
  "isActive": false
}
```

The response contains `id`, `isActive`, and `updatedAt`. Setting `isActive: true` enables the doctor record again, but public discovery still requires the linked user account to be active. A missing doctor returns 404; missing/invalid authentication returns 401 and a disallowed role returns 403.

## 5. Check discovery

In local Swagger, compare the list, detail, and specialty filter. With a private ADMIN account, deactivate a test doctor and confirm they disappear from public discovery and their public detail returns 404. The Doctors service test verifies active-user filtering and stable pagination; M3 API tests cover the administrative role boundary.

<!-- TODO_SCREENSHOT: Swagger doctor discovery and a sanitized ADMIN activation response. -->

