---
title: "5.3.2. Prisma ORM and MySQL"
weight: 2
chapter: false
---

# 5.3.2. Prisma ORM and MySQL

## Objective and technology choices

Prepare schema management and database access for the local backend. MySQL supports transactional relational data, including the relationships among users, doctors, schedules, and appointments.

Prisma keeps the schema, migration history, and generated type-safe client together. NestJS services use Prisma Client through dependency injection rather than handling database access in controllers.

## 1. Inspect the schema and connection setup

The verified schema is `prisma/schema.prisma`. Its `datasource db` uses the `mysql` provider and obtains the connection configuration from the environment. No actual connection value is needed in this report.

The schema contains the eight core entities introduced in [5.2.2. Database Design](../../2-architecture/2-database-design/). Prisma models use names such as `User` and `Patient`, while `@@map` maps them to table names such as `users` and `patients`.

This exact schema fragment defines the generated client and application roles; the remaining schema is omitted:

```prisma
generator client {
  provider      = "prisma-client-js"
  binaryTargets = ["native", "rhel-openssl-3.0.x"]
}

enum Role {
  PATIENT
  DOCTOR
  ADMIN
}
```

The schema's native and Linux binary targets support its existing local and Lambda packaging workflow. This section does not change the schema or repeat the full ERD.

## 2. Follow the development workflow

```text
Prisma schema
→ migration
→ MySQL
→ Prisma Client
→ NestJS service
```

Start with a private connection configuration for a local development database. The following scripts are defined in the backend's `package.json`:

```shell
npm run prisma:validate
npm run prisma:migrate:status
npm run prisma:generate
```

Validation checks the schema. Migration status checks the relationship between the database and migration history. Generation creates the Prisma Client used by TypeScript services.

For an intentional schema change in your development working copy, create and apply a named migration, then regenerate the client:

```shell
npm run prisma:migrate:dev -- --name <descriptive-change>
npm run prisma:generate
```

This matches the README's `npx prisma migrate dev --name <descriptive-change>` workflow. The migration command changes the development database; use it only against the intended local database.

{{< notice warning >}}
Keep the real database connection string private. Check the target database before applying migrations; do not use development migrations or reset operations against a production database.
{{< /notice >}}

## 3. Use Prisma through NestJS

`src/database/prisma.service.ts` defines the shared `PrismaService`. These methods are copied from its class; imports and the class declaration are omitted:

```typescript
async onModuleInit(): Promise<void> {
  await this.$connect();
}

async onModuleDestroy(): Promise<void> {
  await this.$disconnect();
}
```

The service extends `PrismaClient` and implements NestJS lifecycle interfaces. It connects when the module initializes and disconnects when the module is destroyed. For example, `UsersService` injects `PrismaService` to find users and create a PATIENT with the corresponding patient profile.

## 4. Keep migration history consistent

The backend contains `prisma/migrations/` with the existing authentication foundation, core data model, and patient identity/history reason migrations. Track each schema change with Prisma Migrate so local, test, and later AWS database structures can follow the same history.

The verified `migrate:deploy` script runs `prisma migrate deploy` to apply existing migrations during deployment. Detailed RDS connectivity and deployment migration steps belong to the later deployment section.

## 5. Recognize the optional development seed

The backend declares `prisma:seed` and configures `prisma/seed.ts`. The inspected seed control flow checks the production environment and requires an explicit development seed opt-in through `ALLOW_DEV_SEED`.

Seeding is optional development data preparation, separate from schema migration. Keep seed credentials private.

<!-- TODO_SCREENSHOT: a local Prisma migration result, with connection details and credentials hidden. -->
