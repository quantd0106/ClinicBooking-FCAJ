---
title: "5.6. AWS Infrastructure"
weight: 6
chapter: false
---

# AWS Infrastructure

This stage translates the application's requirements into AWS networking, database, storage, secrets, and IAM resources before deployment. The source of truth is the backend repository's `infrastructure/template.yaml`, using the `AWS::Serverless-2016-10-31` transform:

**AWS SAM template → CloudFormation → AWS resources**

The current source declares **31 resources across 19 resource types before the SAM transform**. SAM may generate additional resources; these are source counts, not a count of live AWS resources. The workshop targets **ap-southeast-1 (Asia Pacific — Singapore)**. The template uses the stack's region rather than hard-coding Singapore.

| Component | Purpose |
|---|---|
| VPC / Subnets | Private connectivity for Lambda and RDS across two Availability Zones. |
| Security Groups | Control MySQL and AWS service traffic between workloads. |
| RDS MySQL | Managed relational storage for schedules, appointments, and related data. |
| Secrets Manager | Protect database credentials and JWT signing configuration. |
| VPC Endpoints | Reach S3 and Secrets Manager without a NAT Gateway. |
| S3 | Store private doctor profile files with constrained direct uploads. |
| IAM | Authorize logging, secret access, uploads, and VPC networking. |

## Infrastructure sections

1. [5.6.1. VPC and Network Design](1-vpc-network/)
2. [5.6.2. Security Groups](2-security-groups/)
3. [5.6.3. Amazon RDS for MySQL](3-rds-mysql/)
4. [5.6.4. Secrets Manager and VPC Endpoints](4-secrets-endpoints/)
5. [5.6.5. Private S3 and IAM](5-s3-iam/)

{{< notice info >}}
This section documents the learning/demo infrastructure design only. Deployment, packaging, database migration, live smoke tests, and cleanup are covered later. No AWS resources are created or inspected here.
{{< /notice >}}

<!-- TODO_SCREENSHOT: CloudFormation/VPC resource overview, with sensitive identifiers redacted. -->
