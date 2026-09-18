---
title: "5.3.3. JWT Authentication"
weight: 3
chapter: false
---

# 5.3.3. JWT Authentication

## Objective and verified components

Understand how the backend registers patients, issues JWT access tokens, and authenticates protected requests. The verified implementation uses `AuthController`, `AuthService`, `UsersService`, `JwtStrategy`, and `JwtAuthGuard`.

| Endpoint | Access | Success response |
| --- | --- | --- |
| `POST /api/v1/auth/register` | Public | 201; `id`, `email`, `role`. |
| `POST /api/v1/auth/login` | Public | 200; `accessToken`, `tokenType`, `expiresIn`, and `user` containing `id` and `role`. |
| `GET /api/v1/auth/me` | Bearer JWT | 200; authenticated user's `id`, `email`, `role`. |

These fields follow `src/auth/dto/auth-response.dto.ts`. Safe account responses omit the password hash; the access token is sensitive and must not be published.

## 1. Register a PATIENT account

```text
Client
→ POST /api/v1/auth/register
→ validate RegisterDto
→ hash password with bcrypt
→ create user with role PATIENT and a linked patient profile
→ return id, email, role
```

`RegisterDto` contains only `email` and `password`. Email is trimmed, lowercased, checked as an email, and limited to 191 characters. The password is validated as a string of 12–72 characters.

Public registration creates **PATIENT only**. The client cannot choose DOCTOR or ADMIN. A supplied `role` field is outside the DTO contract and is rejected by the global `ValidationPipe`.

`AuthService.register()` uses `bcrypt.hash()` with the verified `BCRYPT_ROUNDS = 10`. `UsersService.createPatient()` fixes the role and creates the related patient profile in a nested Prisma write. This excerpt is copied from that method; the method wrapper is omitted:

```typescript
const data: Prisma.UserCreateInput = {
  email,
  passwordHash,
  role: Role.PATIENT,
  isActive: true,
  patient: { create: {} },
};
return this.prisma.user.create({ data });
```

The stored value is a password hash, never the plaintext password. A duplicate email produces 409 with `EMAIL_ALREADY_REGISTERED`.

## 2. Log in and obtain an access token

```text
Client
→ POST /api/v1/auth/login
→ validate LoginDto
→ find user by email
→ compare password using bcrypt
→ check that the user is active
→ issue JWT access token
```

`LoginDto` also contains only `email` and `password`; its password validation permits 1–72 characters. `AuthService.login()` compares the submitted password with the stored hash using `bcrypt.compare()`.

The signed JWT payload contains `sub` (the user ID) and `role`. The response includes the Bearer token type and the configured expiry in seconds. JWT signing configuration comes from `ConfigService`; no secret value belongs in the client, source excerpts, or report.

Invalid credentials or an inactive account return 401 with `INVALID_CREDENTIALS`.

## 3. Send an authenticated request

```http
GET /api/v1/auth/me
Authorization: Bearer <token>
```

```text
Client
→ Authorization: Bearer <token>
→ JwtAuthGuard
→ JwtStrategy validates signature and expiry
→ load active user using payload.sub
→ attach authenticated user
→ controller/business logic
```

`JwtStrategy` extracts the Bearer token and sets `ignoreExpiration: false`. Its `validate()` method loads the current database user and returns `id`, `email`, and the current database role. A deleted or inactive user is rejected.

This exact method is taken from `src/auth/jwt-auth.guard.ts`; imports and the surrounding class are omitted:

```typescript
handleRequest<TUser = unknown>(err: unknown, user: TUser | false): TUser {
  if (err || !user) {
    throw new UnauthorizedException({
      code: ApiErrorCode.Unauthorized,
      message: 'Authentication is invalid or has expired.',
    });
  }
  return user;
}
```

The surrounding class extends `AuthGuard('jwt')`. The `/auth/me` handler uses `@UseGuards(JwtAuthGuard)` and obtains the authenticated identity through `@CurrentUser()`.

## 4. Check the authentication contract

Using the local Swagger UI at `/api/docs`, check registration, login, and the protected `/api/v1/auth/me` endpoint with your own private test account. Do not copy account passwords or real tokens into screenshots.

| Situation | Expected response |
| --- | --- |
| Missing, invalid, or expired JWT | 401 `UNAUTHORIZED`. |
| JWT refers to a missing/inactive user | 401 `UNAUTHORIZED`. |
| Invalid login credentials or inactive login account | 401 `INVALID_CREDENTIALS`. |
| Invalid registration/login DTO, including a registration role field | 400 validation error. |

A valid identity without the required role is an authorization failure, covered in [5.3.4](../4-rbac-api-conventions/), and returns 403.

{{< notice warning >}}
Never publish plaintext passwords, password hashes, JWT signing secrets, or real access tokens. Keep any login screenshot sanitized.
{{< /notice >}}

> 📷 Screenshot to add: Swagger Auth endpoints and a successful login response with the access token and personal identifiers hidden.

