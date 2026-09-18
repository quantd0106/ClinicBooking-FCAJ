---
title: "5.6.2. Security Groups"
weight: 2
chapter: false
---

# Security Groups

## Verified rules

The template declares three Security Groups and two separate ingress resources. MySQL ingress is defined by `LambdaToDatabaseIngress`; endpoint HTTPS ingress is defined by `LambdaToAwsServiceEndpointIngress`.

| Security Group | Direction | Port | Source / Destination | Purpose |
|---|---|---|---|---|
| `LambdaSecurityGroup` | Inbound | None declared | — | No application ingress rule on Lambda's VPC interface. |
| `LambdaSecurityGroup` | Outbound | TCP 3306 | Destination: `DatabaseSecurityGroup` | Connect to MySQL privately. |
| `LambdaSecurityGroup` | Outbound | TCP 443 | Destination: `0.0.0.0/0` | HTTPS, subject to available routes and endpoint policies. |
| `DatabaseSecurityGroup` | Inbound | TCP 3306 | Source: `LambdaSecurityGroup` | Accept MySQL connections from the Lambda workload. |
| `DatabaseSecurityGroup` | Outbound | All traffic by creation default | All destinations | No explicit egress is defined in this resource. |
| `AwsServiceEndpointSecurityGroup` | Inbound | TCP 443 | Source: `LambdaSecurityGroup` | Accept requests at the Secrets Manager endpoint ENIs. |
| `AwsServiceEndpointSecurityGroup` | Outbound | All traffic by creation default | All destinations | No explicit egress is defined in this resource. |

CloudFormation adds default outbound allow rules when a newly created Security Group has no explicit egress rules. That applies to the database and endpoint groups here; the Lambda group supplies its own explicit egress rules. See the [CloudFormation Security Group reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-ec2-securitygroup.html).

## Database boundary

The critical rule is **TCP 3306 from the Lambda Security Group**, not a database ingress rule from `0.0.0.0/0`:

```yaml
LambdaToDatabaseIngress:
  Type: AWS::EC2::SecurityGroupIngress
  Properties:
    Description: Lambda to RDS MySQL
    FromPort: 3306
    GroupId:
      Ref: DatabaseSecurityGroup
    IpProtocol: tcp
    SourceSecurityGroupId:
      Ref: LambdaSecurityGroup
    ToPort: 3306
```

An SG reference follows the workload's group membership instead of relying on a fixed Lambda interface IP. This restricts MySQL ingress to workloads associated with `LambdaSecurityGroup`. The rule is combined with private subnet placement and `PubliclyAccessible: false` on RDS.

The Lambda HTTPS egress destination is broader than an SG reference. Private routing still provides no general Internet default route. Endpoint policies and IAM remain separate authorization boundaries. This is the actual template behavior, not a claim that every outbound rule is narrowly scoped.

## Security Group versus Network ACL

| Mechanism | Scope | Connection behavior | Project use |
|---|---|---|---|
| Security Group | Network interfaces / attached resources | Stateful: response traffic for an allowed connection is allowed. | Three explicit groups and their rules. |
| Network ACL | Subnet | Stateless: inbound and outbound traffic are evaluated independently. | No custom Network ACL resource is declared in this template. |

This distinction follows the [Amazon VPC security documentation](https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html). The template's application-specific traffic controls are Security Groups; no custom NACL rules are asserted.

{{< notice warning >}}
Do not broaden database ingress to the Internet. The verified design authorizes Lambda SG → RDS TCP 3306. Default outbound rules on the database and endpoint SGs are a separate hardening consideration.
{{< /notice >}}

<!-- TODO_SCREENSHOT: Security Group rule showing Lambda SG → RDS TCP 3306, with identifiers redacted. -->
