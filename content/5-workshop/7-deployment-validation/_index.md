---
title: "5.7. Deployment and Live Validation"
weight: 7
chapter: false
---

# 5.7. DEPLOYMENT AND LIVE VALIDATION

## Overview

After completing the application and infrastructure, the backend was deployed to AWS using an AWS SAM template processed by AWS CloudFormation. The backend deployment and test documentation records a successful demo deployment in **ap-southeast-1 on 2026-09-09**.

**Client → Amazon API Gateway → AWS Lambda → Amazon RDS for MySQL**

S3 supports private profile uploads, Secrets Manager supplies protected runtime configuration, and CloudWatch receives application and API logs. The infrastructure design is described in [5.6. AWS Infrastructure](../6-aws-infrastructure/).

{{< notice info >}}
The results below come from the historical deployment record in the backend repository. This documentation step does not redeploy the application, call AWS APIs, or rerun live tests. It does not establish the environment's current operational status.
{{< /notice >}}

## Deployment Process

The actual runbook and packaging/maintenance source establish this flow:

1. **Validate the infrastructure template.** Review the SAM template and run the repository's structural infrastructure checks.
2. **Build and package Lambda.** Build the NestJS application, generate the Prisma client, and package the Node.js 24 handler with the required native Prisma runtime and checked-in migrations.
3. **Deploy through reviewed CloudFormation change sets.** The documented process first uses a bootstrap template with the same logical resource names/types and a minimal inline Lambda handler. After the stack-owned private bucket exists, upload the application artifact under its deployment prefix and review a second change set to update the application code.
4. **Provision the runtime and supporting resources.** CloudFormation processes the SAM transform and provisions API Gateway, Lambda, private RDS, S3, networking, IAM, Secrets Manager, endpoints, and logging.
5. **Apply reviewed migrations to private RDS.** The recorded demo invoked the existing Lambda directly with the non-HTTP `deployReviewedMigrations` maintenance action. It applies checked-in migrations with a MySQL advisory lock and Prisma-compatible checksums. This path rejects events containing `requestContext` and is not exposed through API Gateway. The record reports that all three reviewed migrations were applied.
6. **Run live smoke tests and inspect logs.** Verify the application through API Gateway, test the booking lifecycle and private file access, and inspect Lambda/application and API access logs.

These two preparation commands are verified in `package.json` and the deployment runbook. They are shown for reference and were **not executed** during this documentation step:

```bash
npm run infra:validate
npm run build:lambda
```

Migrations require a controlled execution environment with private connectivity to RDS. The deployment record kept RDS private throughout the process. This page does not execute migration, packaging, deployment, or seed operations.

## Deployment Results

| Item | Verified historical result |
|---|---|
| CloudFormation | The deployment record reports `UPDATE_COMPLETE` on 2026-09-09. |
| API Gateway → Lambda | API Gateway served the M1–M5 APIs and Swagger document through the Lambda runtime. |
| Lambda → RDS | Lambda applied reviewed migrations and served database-backed flows against private RDS. |
| Secrets Manager | The documented runtime resolved protected configuration through the Secrets Manager interface endpoint. |
| Private S3 | An authorized DOCTOR presigned-upload flow succeeded; anonymous object access returned HTTP 403. |
| RDS network boundary | The record reports private connectivity; the template sets `PubliclyAccessible: false`. |
| S3 public-access boundary | The template enables all four Block Public Access options; recorded anonymous access failed. |
| Logging | Lambda/application and API access logs were present in CloudWatch. |

These results describe the recorded demo, rather than a fresh check of currently running resources.

<!-- TODO_SCREENSHOT: CloudFormation stack successfully deployed. Redact stack/resource identifiers. -->

<!-- TODO_SCREENSHOT: Private RDS configuration. Redact the endpoint and identifiers. -->

## Live API Validation

The table separates **expected API behavior** from **recorded live evidence**. Expected codes follow the API documentation/controllers. A recorded successful flow does not prove an exact HTTP code when the historical summary does not preserve that code.

| Test | Expected Result | Result |
|---|---|---|
| PATIENT login: `POST /api/v1/auth/login` | HTTP 200. | Patient login passed in the recorded smoke test; exact HTTP code not retained in the summary. |
| GET specialties: `GET /api/v1/specialties` | HTTP 200. | Discovery passed as an aggregate flow; no separate response code is recorded. |
| GET doctors: `GET /api/v1/doctors` | HTTP 200. | Discovery passed as an aggregate flow; no separate response code is recorded. |
| GET available slots: `GET /api/v1/doctors/<doctor-id>/available-slots?date=<future-date>` | HTTP 200. | Slot lookup passed; exact HTTP code not retained in the summary. |
| Create appointment: `POST /api/v1/appointments` | HTTP 201. | Booking passed; the recorded live checks specify HTTP 201. |
| Duplicate booking | HTTP 409, `SLOT_ALREADY_BOOKED`. | Recorded HTTP 409 with `SLOT_ALREADY_BOOKED`. |
| List own appointments: `GET /api/v1/appointments/me` | New booking visible in the caller's authorized appointment list. | Endpoint is documented/implemented; this individual live result is not recorded. |
| Cancel appointment: `PATCH /api/v1/appointments/<appointment-id>/cancel` | HTTP 200, status `CANCELLED`. | Cancellation passed; the summary does not preserve its response code or response body. |
| Recheck/rebook a released slot | Slot can be reused after cancellation, subject to availability and authorization rules. | Slot reuse passed; no individual response code is recorded. |
| DOCTOR login: `POST /api/v1/auth/login` | HTTP 200. | Login endpoint is implemented; a separate DOCTOR login result is not recorded. |
| Presigned-upload request: `POST /api/v1/files/presigned-upload` | Successful DOCTOR request; HTTP 201 follows the POST controller contract. | DOCTOR presigned upload succeeded; exact request response code is not recorded. |
| Direct anonymous private S3 object GET | Access denied. | Recorded HTTP 403. |

The upload implementation returns a **presigned POST URL and form fields** for direct upload to private S3. Its successful flow does not imply public read access.

<!-- TODO_SCREENSHOT: Live booking response, HTTP 201. Redact tokens and private identifiers. -->

<!-- TODO_SCREENSHOT: Duplicate booking, HTTP 409 `SLOT_ALREADY_BOOKED`. Redact private identifiers. -->

## Logging and Monitoring Checks

The historical M5 evidence reports both Lambda/application logs and API access logs in CloudWatch. These were inspected for runtime/application behavior, and the record reports no detected credential, database connection URL, JWT, or presigned-signature patterns.

The template configures API access logging and gives both declared log groups **14-day retention**. Sensitive values must remain out of logs and screenshots. No CloudWatch Alarm deployment is claimed.

<!-- TODO_SCREENSHOT: CloudWatch Lambda logs with sensitive values and private identifiers redacted. -->

## Deployment Issues

The verified quota-related adjustment concerns **reserved Lambda concurrency**. The infrastructure documentation explains that the account quota already limited concurrency and that the required unreserved pool prevented an additional function-level reservation.

The template resolves this with `LambdaReservedConcurrency`: when its value is `0`, the condition selects `AWS::NoValue` and omits `ReservedConcurrentExecutions`. The deployment record confirms the final demo used no function-level reservation and **512 MiB memory**.

The inspected evidence does not establish an earlier memory-quota failure or a CloudFormation rollback sequence. Those incidents are not presented as verified history.

## Final Outcome

The recorded demo deployed the backend successfully and exercised the core M1–M5 APIs through API Gateway and Lambda. Database-backed flows used private RDS; private S3 upload and anonymous-access denial were recorded. Double-booking protection returned `409 SLOT_ALREADY_BOOKED`, and cancellation/slot reuse passed. CloudWatch logging was available.

The SAM/CloudFormation template and repository runbook provide a reproducible infrastructure and deployment process. This remains a **learning/demo environment**; these checks do not establish production readiness.

Evidence sources in the backend repository: `docs/09-deployment.md` (deployment status/runbook), `docs/08-test-plan.md` (recorded M5 evidence), `docs/06-aws-infrastructure.md` (infrastructure/concurrency), and `docs/05-api-design.md` (API contracts), corroborated by the current template, packaging scripts, controllers, Lambda entry, and maintenance source.

## Cost and Cleanup Notes

Running AWS resources may generate costs. RDS instances incur charges while running, and provisioned Secrets Manager interface endpoints incur endpoint-hour and applicable data-processing charges. Refer to [RDS pricing](https://aws.amazon.com/rds/mysql/pricing/) and [AWS PrivateLink pricing](https://aws.amazon.com/privatelink/pricing/).

Use [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) to monitor costs and alert against an appropriate demo budget; no existing budget configuration is asserted. When the demo is no longer required, plan cleanup and review retained buckets, objects, snapshots, and secrets.

{{< notice warning >}}
This page provides no destructive cleanup commands and performs no cleanup. Retained resources can continue to incur charges after stack removal.
{{< /notice >}}
