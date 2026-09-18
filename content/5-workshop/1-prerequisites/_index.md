---
title: "5.1. Prerequisites"
weight: 1
chapter: false
---

# 5.1. PREREQUISITES

## Objective

Before starting the workshop, prepare the local development environment, an AWS account, deployment tools, and the basic AWS permissions needed for deployment. This allows the backend to be developed locally and the serverless infrastructure to be deployed with AWS SAM and AWS CloudFormation.

## Required knowledge

Basic familiarity with the following is sufficient; expert-level knowledge is not required:

- REST APIs.
- Node.js and TypeScript.
- Relational databases and MySQL.
- Git.
- Basic AWS concepts.
- Amazon VPC and Security Group fundamentals.

## Tools to prepare

| Tool | Role |
| --- | --- |
| Visual Studio Code | Edit backend code and infrastructure files. |
| Node.js | Run the backend locally and support the Node.js development workflow. |
| npm | Install and manage project dependencies. |
| Git | Clone the backend source and track changes. |
| GitHub | Host and collaborate on source code. |
| MySQL | Provide the relational database for local development. |
| Prisma CLI | Work with the Prisma schema, migrations, and generated database client. |
| Postman or Swagger UI | Send API requests and inspect the API contract. |
| AWS CLI v2 | Check the AWS identity and region and interact with AWS resources. |
| AWS SAM CLI | Define, build, and deploy the serverless application. |
| Hugo Extended | Build this FCAJ workshop website. |

Hugo Extended is used only for the FCAJ report and workshop website. It is not part of the Clinic Appointment Booking System backend runtime.

## AWS account

1. Prepare an AWS account for the workshop.
2. Avoid root credentials for routine deployment.
3. Use an AWS IAM identity for CLI and deployment operations, with permissions appropriate to the resources defined by SAM/CloudFormation.
4. Configure an AWS CLI profile for that identity.
5. Use ap-southeast-1 — Asia Pacific (Singapore) for this workshop.

Use least-privilege permissions and keep real account identifiers and credentials out of the workshop documentation.

## Environment checks

Check the installed development and deployment tools:

```text
node --version
npm --version
git --version
aws --version
sam --version
```

Check the AWS identity using your configured profile:

```text
aws sts get-caller-identity --profile <your-profile>
```

Check the profile's configured region:

```text
aws configure get region --profile <your-profile>
```

Replace `<your-profile>` with the name of your local AWS CLI profile. Run the identity check locally; its output contains account and identity information that should not be published. The region check should return ap-southeast-1.

## Source code preparation

Clone or download the Clinic Appointment Booking System backend source from the project's available source location. Open the backend project directory and install its dependencies:

```text
npm install
```

Run this command in the backend project, not in the Hugo report repository. This workshop does not assume a public backend repository URL.

## Security notes

{{< notice warning >}}
Never commit .env files. Never hard-code AWS access keys. Never publish database credentials. Use AWS Secrets Manager for sensitive runtime configuration, including database credentials and JWT-related secrets. Do not make Amazon RDS public for convenience.
{{< /notice >}}
