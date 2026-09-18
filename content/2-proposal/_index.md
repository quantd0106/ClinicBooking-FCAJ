---
title: "Proposal"
weight: 2
chapter: false
pre: "<b>2. </b>"
---

# PROJECT PROPOSAL

This proposal describes the Clinic Appointment Booking System, a backend for discovering doctors, viewing available slots, and managing appointments. It presents the business objectives, technical design, implementation milestones, estimated demo costs, and success criteria for deployment on AWS.

## 2.1. Project Overview

Clinic Appointment Booking System is a backend for online clinic appointment booking. It enables patients to discover doctors, view available appointment slots, book appointments, cancel appointments, and manage their appointments.

Doctors can manage their working schedules and update appointment statuses. Administrators support the management of system data within their assigned permissions.

The project provides a practical opportunity to combine backend development, database design, and AWS Cloud deployment in one complete system. The backend uses Node.js, TypeScript, NestJS, Prisma ORM, and MySQL, with JWT authentication, role-based access control (RBAC), and Swagger/OpenAPI documentation.

## 2.2. Problem Statement and Objectives

### Problem statement

- Manual appointment booking can make it difficult to check a doctor's availability.
- Concurrent requests can reserve the same doctor, date, and time unless the backend handles concurrency.
- PATIENT, DOCTOR, and ADMIN need clearly separated permissions.
- The database and file storage need a secure cloud deployment.

### Functional objectives

- Provide authentication.
- Support doctor discovery and schedule management.
- Calculate available appointment slots.
- Support appointment booking and cancellation.
- Manage the appointment status lifecycle and record status history.
- Support private file uploads.

### Technical objectives

- Use JWT authentication and RBAC to protect application access.
- Prevent double booking through MySQL transactions, locking logic, and validation of active appointments.
- Deploy the database privately.
- Deploy the backend on AWS using a serverless architecture.
- Centralize runtime logging.
- Define infrastructure using Infrastructure as Code.

### Booking consistency and status history

PENDING and CONFIRMED appointments are active and block a second booking for the same doctor, date, and time. Cancelled appointments do not block rebooking. A duplicate active booking must return HTTP 409 with the error code SLOT_ALREADY_BOOKED.

The appointment lifecycle supports:

- PENDING → CONFIRMED → COMPLETED
- PENDING → CANCELLED
- CONFIRMED → CANCELLED

Every appointment status change is recorded in appointment_status_history.

## 2.3. Target Users and Main Functions

| Role | Main functions |
| --- | --- |
| PATIENT | Register and log in; view specialties; search for and view doctors; view available slots; create appointments; view personal appointments; cancel appointments. |
| DOCTOR | Log in; manage working schedules; view relevant appointments; update appointment statuses; request presigned URLs for profile or file uploads. |
| ADMIN | Manage specialties; activate or deactivate doctors; perform administrative actions according to system permissions. |

Public registration creates PATIENT accounts only. It does not grant DOCTOR or ADMIN permissions.

JWT authentication identifies the user, while RBAC determines the functions the user is permitted to access. Access to personal or relevant appointment data must respect the user's role and permissions.

## 2.4. Solution Architecture

### Application request flow

The backend runs as a NestJS application on AWS Lambda with a Node.js runtime. Amazon API Gateway receives client requests. NestJS handles authentication, authorization, and business logic, while Prisma ORM accesses Amazon RDS for MySQL.

```text
Client
  → Amazon API Gateway
  → AWS Lambda (NestJS / Node.js)
  → Prisma ORM
  → Amazon RDS for MySQL

AWS Lambda
  ├─ Amazon S3: private objects and presigned URL generation
  ├─ AWS Secrets Manager: sensitive runtime configuration
  └─ Amazon CloudWatch: runtime logs and monitoring

Client
  → Amazon S3: private file upload using a presigned URL
```

Prisma ORM is part of the Lambda application rather than a separate AWS service. The booking operation uses a MySQL transaction and locking logic to prevent concurrent active bookings for the same doctor, date, and time.

### Network isolation and private service access

- Lambda runs inside Amazon VPC.
- RDS uses private subnets and is not publicly accessible.
- The database Security Group allows MySQL port 3306 only from the Lambda Security Group.
- Amazon S3 remains private with Block Public Access enabled. Uploads use presigned URLs.
- Lambda accesses Amazon S3 through an S3 Gateway VPC Endpoint.
- Access to AWS Secrets Manager uses a Secrets Manager Interface VPC Endpoint.
- The final architecture does not include a NAT Gateway.

AWS IAM controls Lambda and deployment permissions. AWS Secrets Manager stores sensitive runtime configuration, including database credentials and JWT-related secret configuration. Amazon CloudWatch receives runtime logs and supports monitoring.

### Deployment configuration

| Component | Design |
| --- | --- |
| Region | ap-southeast-1 — Asia Pacific (Singapore) |
| Database | Amazon RDS for MySQL; db.t4g.micro; Single-AZ; 20 GiB storage; private and non-public |
| Lambda | Node.js runtime; 512 MiB memory; no explicit ReservedConcurrentExecutions in the final deployment |
| Infrastructure | AWS SAM defines, builds, and deploys the serverless infrastructure using AWS CloudFormation. |

## 2.5. AWS Services Used

| Service | Role in the System | Reason for Selection |
| --- | --- | --- |
| Amazon API Gateway | Expose the backend API and forward client requests to Lambda. | Provide a managed API entry point for the serverless backend. |
| AWS Lambda | Run the NestJS backend on a Node.js runtime. | Execute backend logic without managing an EC2 server. |
| Amazon RDS for MySQL | Store users, doctors, schedules, appointments, and status history. | Provide a managed relational database suitable for transactional appointment data and locking logic. |
| Amazon S3 | Store private files and profile uploads using presigned URLs. | Provide object storage suited to uploaded files while keeping the bucket private. |
| Amazon VPC | Isolate Lambda and the private database; support the designed VPC Endpoints. | Control network access to private resources without a NAT Gateway in the final design. |
| AWS IAM | Control permissions for Lambda and deployment operations. | Apply least-privilege access to AWS resources. |
| AWS Secrets Manager | Store database credentials and JWT-related secret configuration. | Keep sensitive runtime configuration out of application code. |
| Amazon CloudWatch | Collect runtime logs and support monitoring. | Centralize application logs and operational visibility. |
| AWS CloudFormation | Provision infrastructure from templates. | Make Infrastructure as Code deployment repeatable and support stack rollback. |
| AWS SAM | Define, build, and deploy the serverless application. | Support the serverless development workflow with CloudFormation-based infrastructure. |

## 2.6. Implementation Plan

| Milestone | Scope | Planned deliverable |
| --- | --- | --- |
| M1 — Foundation / Auth / API Contract | Establish the Node.js, TypeScript, and NestJS foundation; implement JWT authentication and RBAC; define the API contract. | Backend foundation, authentication and authorization flow, and Swagger/OpenAPI contract. |
| M2 — Core Data Model + MySQL | Model the core data with Prisma ORM and MySQL, including appointment status history. | Database schema and data-access foundation. |
| M3 — Doctor Discovery + Schedule | Implement specialty and doctor discovery, working schedules, and available-slot calculation. | APIs for discovering doctors and checking availability. |
| M4 — Appointment Booking Core | Implement booking, cancellation, the status lifecycle, and status-history recording; apply transactions and locking. | Booking logic with active-booking conflict handling and rebooking after cancellation. |
| M5 — AWS Runtime + Infrastructure | Deploy the Lambda runtime, API Gateway, private RDS, private S3, VPC Endpoints, IAM permissions, secrets, and logging using SAM/CloudFormation. | Serverless infrastructure and AWS runtime configuration. |
| M6 — QA + Documentation + Demo | Perform final QA, consolidate documentation, and prepare the demonstration. | QA findings, project documentation, and demo preparation. |

M6 focuses on final QA, documentation, and demonstration preparation. It is not a prerequisite for the technical system to operate. These milestones describe the implementation plan rather than a report of completed live verification.

## 2.7. Estimated Cost

### Demo cost assumptions

The project is intended for learning and demonstration rather than continuous production operation. Cost depends on traffic, resource runtime, storage, and usage.

Reference estimate for a continuously running demo environment: approximately USD 35–40 per month. This is a rough planning range for the supplied design in ap-southeast-1, not a guaranteed monthly AWS bill or a verified pricing calculation. Actual billing may vary.

| Cost component | Consideration |
| --- | --- |
| Amazon RDS for MySQL | The db.t4g.micro Single-AZ instance and 20 GiB storage are a main source of ongoing cost while provisioned. |
| Secrets Manager Interface VPC Endpoint | Endpoint provisioning hours and data processing contribute ongoing cost; cost also depends on the number of Availability Zones used for the endpoint. |
| AWS Lambda and Amazon API Gateway | At demo traffic levels, request and execution costs are expected to be relatively low. |
| Amazon S3 | Uploaded-file storage and requests add usage-based costs. |
| Amazon CloudWatch | Log ingestion, retention, and monitoring usage contribute costs. |
| AWS Secrets Manager | Stored secrets and API usage contribute costs. |

RDS and the Secrets Manager Interface VPC Endpoint are the main continuously billed resources in this design. The other listed usage and storage charges should still be monitored.

### Cost controls

- Configure an AWS Budget for cost tracking.
- Clean up demo resources after the demonstration when they are no longer needed.
- Keep the planned 512 MiB Lambda memory setting under review against actual workload.
- Avoid an unnecessary NAT Gateway; the final architecture intentionally uses the designed VPC Endpoints instead, partly to reduce ongoing cost.

## 2.8. Risks and Mitigation

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Double booking | Two active appointments may conflict for the same doctor, date, and time. | Use a MySQL transaction, locking logic, and validation of PENDING/CONFIRMED appointments; return HTTP 409 SLOT_ALREADY_BOOKED for a duplicate active booking. |
| Unauthorized access | Users may access functions or data outside their permissions. | Apply JWT authentication, RBAC, and least-privilege AWS IAM permissions. |
| Database exposure | Appointment and user data may be exposed through public network access. | Keep RDS private and restrict MySQL port 3306 to the Lambda Security Group. |
| Secret leakage | Database credentials or JWT-related secrets may be disclosed. | Store sensitive configuration in AWS Secrets Manager and avoid hard-coded credentials. |
| Cloud resource cost | Resources left running may create unnecessary charges. | Configure an AWS Budget, clean up after the demo, and avoid an unnecessary NAT Gateway. |
| AWS account quota limits | Deployment or runtime operation may be limited by available quotas. | Validate quotas and select suitable Lambda memory and concurrency settings; the final deployment uses 512 MiB memory without explicit ReservedConcurrentExecutions. |
| Deployment failure | An incomplete deployment may leave the environment unavailable or inconsistent. | Use AWS CloudFormation rollback, review logs, and validate the configuration before redeployment. |

## 2.9. Expected Results and Success Criteria

### Expected result

The expected outcome is a working serverless backend for the Clinic Appointment Booking System deployed on AWS.

### Proposal success criteria

- User authentication works and access follows the assigned role.
- Patients can discover doctors and view available appointment slots.
- Booking succeeds for an available slot.
- A duplicate active booking returns HTTP 409 with SLOT_ALREADY_BOOKED.
- A cancelled appointment releases the slot for rebooking.
- The status lifecycle supports PENDING → CONFIRMED → COMPLETED, PENDING → CANCELLED, and CONFIRMED → CANCELLED.
- Every appointment status change is recorded in appointment_status_history.
- Amazon RDS for MySQL is not publicly accessible.
- The Amazon S3 bucket remains private.
- Lambda can access the required private resources through the designed network configuration.
- Amazon CloudWatch receives application logs.
- Infrastructure can be deployed using AWS SAM and AWS CloudFormation.

These are proposed acceptance criteria, not a live test report. Detailed final verification and evidence will be documented later in the Workshop section.
