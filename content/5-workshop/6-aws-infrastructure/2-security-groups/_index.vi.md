---
title: "5.6.2. Security Groups"
weight: 2
chapter: false
---

# Security Groups

## Các rule đã đối chiếu

Template khai báo ba Security Groups và hai resource ingress riêng. MySQL ingress được định nghĩa bởi `LambdaToDatabaseIngress`; HTTPS ingress cho endpoint được định nghĩa bởi `LambdaToAwsServiceEndpointIngress`.

| Security Group | Hướng | Cổng | Nguồn / Đích | Mục đích |
|---|---|---|---|---|
| `LambdaSecurityGroup` | Inbound | Không khai báo | — | Không mở application ingress trên network interface VPC của Lambda. |
| `LambdaSecurityGroup` | Outbound | TCP 3306 | Đích: `DatabaseSecurityGroup` | Kết nối MySQL qua mạng riêng. |
| `LambdaSecurityGroup` | Outbound | TCP 443 | Đích: `0.0.0.0/0` | HTTPS, phụ thuộc route và endpoint policy có sẵn. |
| `DatabaseSecurityGroup` | Inbound | TCP 3306 | Nguồn: `LambdaSecurityGroup` | Nhận kết nối MySQL từ workload Lambda. |
| `DatabaseSecurityGroup` | Outbound | Mọi lưu lượng theo mặc định lúc tạo | Mọi đích | Resource không định nghĩa egress tường minh. |
| `AwsServiceEndpointSecurityGroup` | Inbound | TCP 443 | Nguồn: `LambdaSecurityGroup` | Nhận request tại ENI của endpoint Secrets Manager. |
| `AwsServiceEndpointSecurityGroup` | Outbound | Mọi lưu lượng theo mặc định lúc tạo | Mọi đích | Resource không định nghĩa egress tường minh. |

CloudFormation thêm outbound allow rule mặc định khi Security Group mới tạo không có egress rule tường minh. Điều này áp dụng cho group database và endpoint tại đây; group Lambda có các egress rule riêng được khai báo rõ. Tham khảo [CloudFormation Security Group reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-ec2-securitygroup.html).

## Ranh giới truy cập database

Rule quan trọng là **TCP 3306 từ Lambda Security Group**, không phải database ingress từ `0.0.0.0/0`:

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

Tham chiếu SG dựa trên membership của workload thay vì cố định IP network interface của Lambda. Cách này giới hạn MySQL ingress cho workload thuộc `LambdaSecurityGroup`. Rule được kết hợp với private subnet và cấu hình `PubliclyAccessible: false` của RDS.

Đích HTTPS egress của Lambda rộng hơn một tham chiếu SG. Private routing vẫn không cung cấp default route Internet tổng quát. Endpoint policy và IAM là các ranh giới cấp quyền riêng. Đây là hành vi thực tế của template, không phải khẳng định mọi outbound rule đều được giới hạn chặt.

## Security Group và Network ACL

| Cơ chế | Phạm vi | Hành vi kết nối | Sử dụng trong dự án |
|---|---|---|---|
| Security Group | Network interface / tài nguyên được gắn | Stateful: cho phép lưu lượng phản hồi của kết nối đã được cho phép. | Ba group tường minh cùng các rule. |
| Network ACL | Subnet | Stateless: đánh giá inbound và outbound độc lập. | Template không khai báo resource Network ACL tùy chỉnh. |

Phân biệt này theo [tài liệu bảo mật Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html). Các kiểm soát lưu lượng riêng cho ứng dụng trong template dùng Security Groups; không khẳng định có custom NACL rule.

{{< notice warning >}}
Không mở rộng database ingress ra Internet. Thiết kế đã đối chiếu cấp quyền Lambda SG → RDS TCP 3306. Outbound rule mặc định của SG database và endpoint là vấn đề hardening riêng.
{{< /notice >}}

<!-- TODO_SCREENSHOT: Rule Security Group thể hiện Lambda SG → RDS TCP 3306, đã che các định danh. -->
