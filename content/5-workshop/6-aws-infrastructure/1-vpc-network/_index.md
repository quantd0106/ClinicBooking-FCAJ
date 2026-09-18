---
title: "5.6.1. VPC and Network Design"
weight: 1
chapter: false
---

# VPC and Network Design

## Resources and subnet placement

`ClinicVpc` enables DNS support and DNS hostnames. Its CIDR comes from a template parameter. The template selects Availability Zone indices **0 and 1** using `Fn::GetAZs` and `Fn::Select`; AZ names depend on the stack region.

| Subnet | AZ selection | Route table | Workload placement |
|---|---|---|---|
| `PublicSubnetA` | Index 0 | `PublicRouteTable` | No Lambda or RDS placement declared here. |
| `PublicSubnetB` | Index 1 | `PublicRouteTable` | No Lambda or RDS placement declared here. |
| `PrivateSubnetA` | Index 0 | `PrivateRouteTable` | Lambda, RDS subnet group, Secrets Manager endpoint ENI. |
| `PrivateSubnetB` | Index 1 | `PrivateRouteTable` | Lambda, RDS subnet group, Secrets Manager endpoint ENI. |

All four subnet resources set `MapPublicIpOnLaunch: false`. Lambda and RDS **share the same two private subnets**; the template does not create separate application and database subnet pairs.

`InternetGatewayAttachment` attaches `InternetGateway` to the VPC. `PublicDefaultRoute` routes `0.0.0.0/0` through that gateway. `PublicSubnetARouteTableAssociation` and `PublicSubnetBRouteTableAssociation` connect the public subnets to `PublicRouteTable`. The corresponding `PrivateSubnetARouteTableAssociation` and `PrivateSubnetBRouteTableAssociation` connect both private subnets to `PrivateRouteTable`.

The private route table has **no Internet default route and no NAT Gateway**. Its S3 service path is provided by `S3GatewayEndpoint`. `SecretsManagerEndpoint` creates private network interfaces in both private subnets and enables private DNS.

## Application traffic

{{< mermaid >}}
graph TD
  Client["Internet / Client"] --> API["API Gateway HTTP API"]
  API --> Lambda
  subgraph VPC
    Lambda["Lambda - PrivateSubnetA/B"]
    RDS["RDS MySQL - private DB subnet group"]
    S3EP["S3 Gateway Endpoint - PrivateRouteTable"]
    SecretsEP["Secrets Manager Interface Endpoint - private ENIs"]
    Lambda -->|TCP 3306| RDS
    Lambda -->|HTTPS| S3EP
    Lambda -->|HTTPS 443| SecretsEP
  end
  S3EP --> S3["Private S3 bucket"]
  SecretsEP --> Secrets["Secrets Manager"]
{{< /mermaid >}}

API Gateway invokes Lambda through the AWS service integration. This does not require inbound HTTP rules on Lambda's VPC Security Group. Lambda's VPC attachment gives application code private access to MySQL and the designed service endpoints.

The actual `ClinicApiFunction` VPC configuration is:

```yaml
VpcConfig:
  SecurityGroupIds:
    - Ref: LambdaSecurityGroup
  SubnetIds:
    - Ref: PrivateSubnetA
    - Ref: PrivateSubnetB
```

`DatabaseSubnetGroup` also references `PrivateSubnetA` and `PrivateSubnetB`. A DB subnet group spanning two AZs does not make the database Multi-AZ: the DB instance explicitly uses Single-AZ.

The function uses `nodejs24.x`, 512 MiB memory, a 30-second timeout, and `x86_64`. Its `ReservedConcurrentExecutions` property is conditional: parameter `LambdaReservedConcurrency` defaults to zero, which selects `AWS::NoValue` and omits the property. A positive parameter enables an explicit limit. This is template behavior, not a verification of deployed concurrency settings.

## Why this design

RDS is private so application data is reached through controlled internal connections rather than a public database endpoint. Lambda uses the same VPC to reach RDS on TCP 3306. S3 and Secrets Manager are the runtime AWS service paths covered by the two endpoints.

Omitting NAT avoids adding a NAT Gateway to this demo design. It also limits connectivity: normal runtime must not depend on arbitrary outbound Internet services. A Security Group rule permitting HTTPS does not supply a missing Internet route. Adding another external dependency would require a separate network design decision.

Lambda execution logs are delivered through the Lambda logging service; the template does not declare an additional CloudWatch Logs VPC endpoint.

{{< notice info >}}
There is no NAT Gateway in this template. The Internet Gateway and public route do not give the private Lambda subnets general Internet access.
{{< /notice >}}

<!-- TODO_SCREENSHOT: VPC subnets, route-table associations, and endpoint overview, with identifiers redacted. -->
