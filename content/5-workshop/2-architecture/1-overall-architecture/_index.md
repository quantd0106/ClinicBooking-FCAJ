---
title: "5.2.1. Overall Architecture"
weight: 1
chapter: false
---

# 5.2.1. Overall Architecture

## Main request path

```text
Client
→ Amazon API Gateway
→ AWS Lambda
→ NestJS
→ Prisma ORM
→ Amazon RDS for MySQL
```

Amazon API Gateway provides the public HTTP entry point. AWS Lambda runs the Node.js backend, with NestJS handling authentication, RBAC, and application logic. Prisma ORM is part of that backend and translates data-access operations into MySQL queries; it is not a separate AWS service.

The backend uses Node.js, TypeScript, NestJS, Prisma ORM, and MySQL. JWT identifies authenticated users, RBAC separates PATIENT, DOCTOR, and ADMIN permissions, and Swagger/OpenAPI documents the API contract.

Lambda avoids the need to operate a permanent EC2 backend server. Amazon RDS for MySQL provides a managed relational database suited to transactional appointment data, including the transaction and locking logic used to prevent double booking.

## Architecture diagram

The VPC boundary represents Amazon VPC. Lambda is attached to private application subnets, while RDS is located in private database subnets. The endpoints shown provide the designed paths to Amazon S3 and AWS Secrets Manager.

{{< mermaid >}}
graph TD
  Client["Client"] --> API["Amazon API Gateway"]
  API --> Backend
  subgraph VPC
    Backend["AWS Lambda / NestJS"]
    Database["Amazon RDS for MySQL"]
    S3Endpoint["S3 Gateway VPC Endpoint"]
    SecretsEndpoint["Secrets Manager Interface Endpoint"]
    Backend -->|Prisma ORM - TCP 3306| Database
    Backend --> S3Endpoint
    Backend --> SecretsEndpoint
  end
  S3Endpoint --> Storage["Amazon S3 - private bucket"]
  SecretsEndpoint --> Secrets["AWS Secrets Manager"]
  Backend --> Logs["Amazon CloudWatch"]
{{< /mermaid >}}

## Supporting service paths

| Path | Role |
| --- | --- |
| Lambda → AWS Secrets Manager | Read sensitive runtime configuration, including database credentials and JWT-related secret configuration, without placing secrets in source code. |
| Lambda → Amazon S3 | Support private file and profile uploads using presigned URLs. Amazon S3 is appropriate for object and file storage. |
| Lambda → Amazon CloudWatch | Provide runtime logs and monitoring for application observability. |

AWS IAM controls Lambda and deployment permissions. AWS SAM defines, builds, and deploys the serverless infrastructure through AWS CloudFormation, making the infrastructure repeatable.

## Networking and access rules

- Lambda is attached to private VPC subnets.
- RDS uses private database subnets and is not publicly accessible.
- The RDS Security Group accepts MySQL TCP 3306 only from the Lambda Security Group.
- The Amazon S3 bucket is private with Block Public Access enabled.
- Lambda accesses Amazon S3 through an S3 Gateway VPC Endpoint.
- Lambda accesses AWS Secrets Manager through a Secrets Manager Interface Endpoint.
- The final architecture has no NAT Gateway.

Private RDS and the restricted database Security Group reduce direct database exposure. The two VPC Endpoints provide the designed service access without adding a NAT Gateway.

## Deployment configuration

| Component | Configuration |
| --- | --- |
| Region | ap-southeast-1 — Asia Pacific (Singapore) |
| RDS | MySQL; db.t4g.micro; Single-AZ; 20 GiB storage; private; PubliclyAccessible = false |
| Lambda | Node.js runtime; 512 MiB memory; no explicit reserved concurrency in the final deployment |
| Infrastructure | AWS SAM and AWS CloudFormation |

This RDS design is Single-AZ; it does not claim Multi-AZ database deployment.
