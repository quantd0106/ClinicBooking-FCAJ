---
title: "5.3.1. NestJS Project Setup"
weight: 1
chapter: false
---

# 5.3.1. NestJS Project Setup

## Objective and framework choice

Prepare the existing NestJS backend for local development. NestJS combines TypeScript, feature modules, and dependency injection, making it easier to extend the API without putting database access and business rules into controllers.

Guards handle authentication and authorization, Pipes validate incoming data, and Interceptors handle cross-cutting concerns such as request logging. The project follows Controller → Service/domain layer → Prisma.

## 1. Prepare the existing project

Clone or download the backend from the project source available to you, then open its directory in Visual Studio Code. Confirm that it contains `package.json`, `src/`, and `prisma/schema.prisma`. This workshop continues that project rather than generating a different NestJS scaffold.

Install the dependencies declared by the backend:

```shell
npm install
```

The verified dependencies include NestJS 11, TypeScript, Prisma Client and Prisma CLI 6.12.0, `@nestjs/config`, `@nestjs/swagger`, `class-validator`, `class-transformer`, Passport JWT, and bcrypt.

## 2. Prepare environment configuration

`src/app.module.ts` registers a global `ConfigModule` with caching and `environmentValidationSchema`. `ConfigService` supplies configuration to the application, database connection, and JWT setup.

Prepare private local configuration according to the backend documentation. The following are configuration placeholders, not usable values or a complete configuration file:

```text
DATABASE_URL=<mysql-connection-string>
JWT_SECRET=<secure-secret>
PORT=<local-port>
```

The project also validates `NODE_ENV`, `JWT_EXPIRES_IN_SECONDS`, and `APP_TIMEZONE`. The clinic/presentation timezone is `Asia/Ho_Chi_Minh`. Never copy real connection strings or signing secrets into this report.

{{< notice warning >}}
Keep local configuration private. Never commit .env files, database credentials, JWT signing secrets, or AWS access keys.
{{< /notice >}}

## 3. Understand bootstrap and the API prefix

`src/main.ts` calls `createNestApplication()` and listens on the configured `PORT`. `src/app.factory.ts` creates the Nest application, then calls `configureApplication()` and `configureSwagger()`.

This excerpt is copied from `src/bootstrap.ts`; imports, its wrapper function, and the filter/interceptor setup are omitted:

```typescript
app.setGlobalPrefix('api/v1');
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,
    whitelist: true,
    forbidNonWhitelisted: true,
    forbidUnknownValues: true,
  }),
);
```

The public API prefix is `/api/v1`. The global Pipe transforms DTO input and rejects fields outside the validated DTO contract. The same bootstrap installs `HttpExceptionFilter` and `HttpLoggingInterceptor`.

## 4. Locate the modules

| Module | Verified source | Responsibility |
| --- | --- | --- |
| Auth | `src/auth/auth.module.ts` | Registration, login, JWT strategy, and authentication guard. |
| Users | `src/users/users.module.ts` | User lookup and PATIENT creation; imported by AuthModule. |
| Database | `src/database/database.module.ts` | Shared Prisma access and persistence services. |
| Common | `src/common/common.module.ts` | Shared application support; related code includes authorization, ownership, time, and error handling. |
| Specialties | `src/specialties/specialties.module.ts` | Medical specialty API. |
| Doctors | `src/doctors/doctors.module.ts` | Doctor discovery and profile API. |
| Schedules | `src/schedules/schedules.module.ts` | Doctor working schedules and available slots. |
| Appointments | `src/appointments/appointments.module.ts` | Appointment operations. |
| Files | `src/files/files.module.ts` | Private profile-file operations. |

The feature modules already exist. This foundation section explains how they fit together; their later workflows are covered separately.

## 5. Check the local backend

After private configuration and the database preparation in [5.3.2](../2-prisma-mysql/), use the scripts verified in `package.json`:

```shell
npm run start:dev
```

Run the following checks separately; the development server keeps running until you stop it:

```shell
npm run build
npm run lint
```

`start:dev` runs `nest start --watch`, `build` runs `nest build`, and `lint` runs ESLint across `src`, `test`, and Prisma TypeScript files.

`src/swagger.ts` registers Swagger UI at `/api/docs`, with Bearer authentication support. Open `http://localhost:<local-port>/api/docs` using the port configured for the backend. Swagger is separate from the Hugo site on port 1313.

<!-- TODO_SCREENSHOT: backend project folders and Swagger UI showing the Auth endpoints. -->

