---
title: "5.3.4. RBAC and API Conventions"
weight: 4
chapter: false
---

# 5.3.4. RBAC and API Conventions

## 1. Separate authentication and authorization

Authentication answers **“Who is the user?”** Authorization answers **“What is this authenticated user allowed to do?”**

The project has exactly three roles: `PATIENT`, `DOCTOR`, and `ADMIN`. JWT authentication establishes the current user identity. Role checks and resource ownership checks then enforce permissions.

```text
JwtAuthGuard
→ authenticated identity
→ RolesGuard checks required roles
→ controller
→ service applies business and ownership rules
```

## 2. Follow the verified guard/decorator pattern

`src/common/auth/roles.decorator.ts` defines the actual `Roles` decorator. This excerpt omits imports only:

```typescript
export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]): MethodDecorator & ClassDecorator =>
  SetMetadata(ROLES_KEY, roles);
```

`RolesGuard` reads this metadata from the handler or controller using `Reflector.getAllAndOverride()`. If required roles are configured, it checks for an authenticated `request.user` and then checks whether that user's role is allowed.

For an actual example, `src/appointments/appointments.controller.ts` applies `@UseGuards(JwtAuthGuard, RolesGuard)` at class level and `@Roles(Role.PATIENT)` to its create handler. The following original method excerpt omits Swagger decorators; the two comments indicate omitted surrounding code:

```typescript
// Class-level decorators:
@UseGuards(JwtAuthGuard, RolesGuard)
// Method-level decorators on create():
@Post()
@Roles(Role.PATIENT)
```

This is an excerpt of decorators, not a standalone compilable controller. Together they restrict `POST /api/v1/appointments` to an authenticated PATIENT. Detailed booking behavior belongs to the later Appointment section.

Passing a role check does not automatically grant access to every resource. The project's ownership checks remain necessary in the service layer.

## 3. Distinguish 401 from 403

| Response | Meaning | Project example |
| --- | --- | --- |
| 401 Unauthorized | Authentication has not succeeded. | Missing/invalid/expired token; invalid login credentials. |
| 403 Forbidden | The authenticated identity lacks permission. | Role not permitted by `RolesGuard`, or a resource ownership failure. |

`JwtAuthGuard` runs before `RolesGuard` in the example. `RolesGuard` also defensively returns 401 if required roles exist but there is no user, and returns 403 when the user's role is not allowed.

## 4. Use the API prefix and time policy

All business API routes use the `/api/v1` prefix configured in `src/bootstrap.ts`. Swagger UI is separately registered at `/api/docs`.

Use ISO-8601 for date/time instants where applicable, preferably UTC with a `Z` suffix. Backend computations use UTC; the clinic/presentation timezone is `Asia/Ho_Chi_Minh`.

There is an important distinction in the actual implementation:

- Schedule and appointment calendar fields represent the clinic date and time, with formats such as `YYYY-MM-DD` and `HH:mm`.
- `ClinicTimeService` interprets those calendar values using `APP_TIMEZONE` and converts them to UTC instants for comparisons and validation.
- UTC-oriented processing does not mean that a separate MySQL date or time field is already a complete UTC instant. Keep the clinic timezone when combining those values.

This follows `src/common/time/clinic-time.service.ts` and the backend API/database documentation.

## 5. Follow HTTP response conventions

| Status | Use |
| --- | --- |
| 200 OK | Successful retrieval or login. |
| 201 Created | Successful creation, including registration. |
| 400 Bad Request | Invalid DTO input or invalid domain input. |
| 401 Unauthorized | Missing or invalid authentication. |
| 403 Forbidden | Role or resource permission denied. |
| 404 Not Found | Requested resource does not exist. |
| 409 Conflict | Conflicts such as an existing email or an occupied booking slot. |

Later appointment conflicts use 409 and a domain code such as `SLOT_ALREADY_BOOKED`. This foundation section does not implement or repeat the detailed booking transaction.

## 6. Read the standardized error response

The global `HttpExceptionFilter` returns exactly `statusCode`, `code`, and `message`. The following line is copied from `src/common/errors/http-exception.filter.ts`:

```typescript
response.status(statusCode).json({ statusCode, code, message });
```

A safe example matching the verified invalid-credentials DTO is:

```json
{
  "statusCode": 401,
  "code": "INVALID_CREDENTIALS",
  "message": "Invalid email or password."
}
```

The filter preserves a domain code when supplied, otherwise selects a status-based default. Validation message arrays are joined into one string separated by semicolons. Unhandled errors use HTTP 500 and a generic message instead of exposing internal exception details.

## Completion check

Confirm that local Swagger exposes the Auth endpoints, public registration cannot select a privileged role, protected routes require a Bearer token, and role failures are distinct from authentication failures. Keep any real tokens, credentials, and personal data out of the report.

