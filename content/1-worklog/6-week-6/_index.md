---
title: "Week 6 - CloudWatch, CloudTrail and Clinic Booking Project"
weight: 6
chapter: false
pre: "<b>1.6. </b>"
---

# WORKLOG WEEK 6

**Date range:** 07/09/2026 - 13/09/2026

## Week 6 Objectives

- Study Amazon CloudWatch and AWS CloudTrail.
- Learn about logs, metrics, and alarms for monitoring AWS resources.
- Consolidate the AWS knowledge learned during the previous weeks.
- Begin applying that knowledge to the Clinic Appointment Booking System project.
- Identify a suitable cloud architecture for the project.

## Tasks Completed During the Week

| Day | Task | Start Date | Completion Date | Reference |
| --- | --- | --- | --- | --- |
| 2 - 3 | Study Amazon CloudWatch and AWS CloudTrail | 07/09/2026 | 08/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| 4 | Monitor logs, metrics, and resource status with Amazon CloudWatch | 09/09/2026 | 09/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| 5 | Study CloudWatch Logs, metrics, alarms, and application error monitoring on AWS | 10/09/2026 | 10/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| 6 - 7 | Consolidate knowledge and begin building the Clinic Appointment Booking System | 11/09/2026 | 12/09/2026 | — |
| 7 - Sunday | Study the AWS deployment architecture for the Clinic Appointment Booking System | 12/09/2026 | 13/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |

### System implementation scope

- Backend using Node.js, TypeScript, and NestJS.
- MySQL database with Prisma ORM.
- JWT authentication.
- PATIENT / DOCTOR / ADMIN authorization.
- Management of specialties, doctors, and working schedules.
- Appointment booking flow.

### Proposed AWS architecture

```text
Client
→ Amazon API Gateway
→ AWS Lambda
→ Amazon RDS for MySQL
```

The planned architecture also considers Amazon S3, Amazon VPC, Security Groups, AWS IAM, AWS Secrets Manager, and Amazon CloudWatch.

## Week 6 Results

- Understand the basic roles of Amazon CloudWatch and AWS CloudTrail.
- Use logs and metrics to monitor AWS resource activity.
- Consolidate and apply the AWS knowledge learned to a practical problem.
- Begin building the backend of the Clinic Appointment Booking System.
- Identify a Serverless-oriented AWS architecture for the system.
- Establish a foundation for detailed project implementation in the Workshop section.
