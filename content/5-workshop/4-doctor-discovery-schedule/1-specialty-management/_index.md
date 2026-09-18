---
title: "5.4.1. Specialty Management"
weight: 1
chapter: false
---

# 5.4.1. Specialty Management

## 1. Understand the specialty catalog

A specialty represents an area of medical practice. Keeping it separate from the doctor profile avoids repeating catalog data and supports the many-to-many relationship through `doctor_specialties`: a doctor can be associated with multiple specialties, and a specialty can be associated with multiple doctors.

The implementation is in `src/specialties/specialties.controller.ts`, `specialties.service.ts`, and `dto/specialty.dto.ts`.

## 2. Identify endpoints and access

| Method | Endpoint | Purpose | Access |
| --- | --- | --- | --- |
| GET | `/api/v1/specialties` | List/search active specialties; 200. | Public; no JWT required. |
| POST | `/api/v1/specialties` | Create a specialty; 201. | Bearer JWT, ADMIN. |
| PATCH | `/api/v1/specialties/:id` | Update specialty fields; 200. | Bearer JWT, ADMIN. |
| DELETE | `/api/v1/specialties/:id` | Deactivate a specialty; 204, no response body. | Bearer JWT, ADMIN. |

Write handlers use `JwtAuthGuard`, `RolesGuard`, and `@Roles(Role.ADMIN)`. DELETE sets `isActive: false`; it does not physically remove the specialty or its doctor associations.

## 3. Search and paginate

`SpecialtyQueryDto` accepts the optional `search` query parameter, trimmed and limited to 100 characters. The shared pagination DTO supplies `page` (default 1, minimum 1) and `pageSize` (default 20, range 1–100).

```http
GET /api/v1/specialties?search=cardio&page=1&pageSize=20
```

Search uses a `contains` condition on the specialty name. Results always require `isActive: true`, ordered by name and then ID. The service obtains the page and count in a Prisma transaction.

The following exact excerpt from `SpecialtiesService.list()` omits surrounding query and response code:

```typescript
const where: Prisma.SpecialtyWhereInput = {
  isActive: true,
  ...(query.search ? { name: { contains: query.search } } : {}),
};
```

The response envelope contains `items`, `page`, `pageSize`, `totalItems`, and `totalPages`. Each item contains `id`, `name`, `description`, and `isActive`. No search case-sensitivity guarantee is added beyond the actual database query.

## 4. Create and update using the verified DTO

A safe create request body is:

```json
{
  "name": "Cardiology",
  "description": "Heart and circulatory system care."
}
```

`CreateSpecialtyDto` accepts a trimmed `name` of 2–100 characters and an optional `description` of up to 2000 characters. Description may be null; the service stores an empty description as null.

`UpdateSpecialtyDto` makes those fields optional and adds optional boolean `isActive`. Send at least one field; an empty PATCH body returns 400 `VALIDATION_ERROR`.

Duplicate names return 409 `SPECIALTY_ALREADY_EXISTS`. A well-formed UUID referencing a missing specialty returns 404 `NOT_FOUND`; malformed route IDs and invalid DTO input return 400. Authentication and role errors follow the 401/403 rules from 5.3.

## 5. Check in local Swagger

Use `/api/docs` to browse without a token, then check catalog writes with your private ADMIN test account. Verify that a deactivated specialty disappears from public listing. The historical `search=cardio` HTTP 200 check is consistent with this query contract; results depend on your local data.

<!-- TODO_SCREENSHOT: Swagger Specialty endpoints and a sanitized search result. -->

