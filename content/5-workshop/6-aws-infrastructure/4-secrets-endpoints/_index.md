---
title: "5.6.4. Secrets Manager and VPC Endpoints"
weight: 4
chapter: false
---

# Secrets Manager and VPC Endpoints

## Secret ownership and references

Sensitive runtime configuration must not be hard-coded in the application or embedded as plaintext in normal Lambda environment configuration. The template uses two secret sources:

| Source | Ownership | Runtime reference |
|---|---|---|
| Database credentials | Managed by RDS through `ManageMasterUserPassword: true`; no separate DB secret resource is declared. | `ClinicDatabase.MasterUserSecret.SecretArn`, supplied through `DATABASE_SECRET_ARN`. |
| JWT signing configuration | `JwtSecret`, an `AWS::SecretsManager::Secret`. | Symbolic `Ref: JwtSecret`, supplied through `JWT_SECRET_ARN`. |

The conceptual protected values are `<database-credentials>` and `<jwt-secret>`. No real secret value is needed to explain this design.

`DB_HOST`, `DB_NAME`, and `DB_PORT` provide database configuration metadata; their actual deployment values are not reproduced here. Normal template environment configuration contains secret references, not a plaintext database connection string or JWT signing secret.

## Lambda startup behavior

`src/lambda.ts` calls `resolveLambdaRuntimeConfiguration` before importing and initializing the NestJS application. In `src/aws/runtime-secrets.ts`, the resolver reads the referenced secrets through the Secrets Manager client when the corresponding runtime values are absent.

It constructs the encoded Prisma database connection configuration in process memory, using the DB metadata and protected credentials, with a connection limit of 2 and pool/connect timeouts of 5 seconds. It also supplies the JWT configuration in memory before application initialization. The initialization promise and Lambda handler are cached for reuse in an existing execution environment.

This is startup resolution, not secret retrieval on every API request. The source does not implement per-request cache refresh after secret rotation; do not assume it does.

`RuntimeSecrets` in the execution role grants `secretsmanager:GetSecretValue` for the two symbolic secret references only:

```yaml
Effect: Allow
Action:
  - secretsmanager:GetSecretValue
Resource:
  - Fn::GetAtt:
      - ClinicDatabase
      - MasterUserSecret.SecretArn
  - Ref: JwtSecret
```

The interface endpoint policy also limits this action to those two secrets. Its `Principal: '*'` does not independently authorize callers: the IAM role and endpoint policy must both permit the access.

## Two endpoint types

| Endpoint | Type / mechanism | Actual placement | Access boundary |
|---|---|---|---|
| `S3GatewayEndpoint` | Gateway; route-table based S3 service path. | Associated with `PrivateRouteTable`. | Endpoint policy allows `s3:GetObject` and `s3:PutObject` for the bucket's `doctors/*` objects; IAM is still required. |
| `SecretsManagerEndpoint` | Interface; PrivateLink private network interfaces with private DNS. | ENIs in `PrivateSubnetA/B`. | Endpoint SG accepts TCP 443 from Lambda SG; endpoint policy scopes secret reads. |

A short excerpt from the interface endpoint resource is below; its policy is omitted from this excerpt:

```yaml
SecretsManagerEndpoint:
  Type: AWS::EC2::VPCEndpoint
  Properties:
    PrivateDnsEnabled: true
    SecurityGroupIds:
      - Ref: AwsServiceEndpointSecurityGroup
    ServiceName:
      Fn::Sub: com.amazonaws.${AWS::Region}.secretsmanager
    SubnetIds:
      - Ref: PrivateSubnetA
      - Ref: PrivateSubnetB
    VpcEndpointType: Interface
    VpcId:
      Ref: ClinicVpc
```

Private DNS allows the Secrets Manager SDK service name to resolve through this endpoint. With no NAT and no private Internet default route, this is the designed path for Lambda's secret reads. The S3 gateway path uses routing instead of an interface endpoint SG.

## Cost and retention notes

An interface endpoint is billed for provisioned endpoint hours in each AZ, plus applicable data processing; charges do not depend only on API request activity. See [AWS PrivateLink pricing](https://aws.amazon.com/privatelink/pricing/). No dollar amount is estimated here.

`JwtSecret` has `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain`. Its retention must be considered in later cleanup planning. Database credentials have RDS-managed ownership; this page does not retrieve or manipulate either secret.

{{< notice warning >}}
The Secrets Manager interface endpoint can incur ongoing charges while provisioned. This demo uses endpoints instead of NAT for its required AWS paths. Detailed cost and cleanup guidance comes later.
{{< /notice >}}

<!-- TODO_SCREENSHOT: Secrets Manager VPC Endpoint type, private DNS, subnets, and SG attachment. Do not show secret values. -->
